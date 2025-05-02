#codex/upload #codex/download

| related | [[Codex Blocks]], [[Block Storage]], [[Codex Block Exchange Protocol]] |
| ------- | ---------------------------------------------------------------------- |

For a high level overview of how Codex works, you can check the [[Codex-BitTorrent Integration presentation]] slides.

To have a more detailed overview of the block exchange architecture, please refer to [Codex Block Exchange Architecture](https://link.excalidraw.com/readonly/L0Rz0LU3oUBDjpHh9rIp).

We upload the content with API `/api/codex/v1/data`. The handler defined in `codex/rest/api.nim` calls `CodexNodeRef.store` and then returns the `Cid` of the manifest file corresponding to the contents:

```nim
without cid =? (
  await node.store(
    AsyncStreamWrapper.new(reader = AsyncStreamReader(reader)),
	  filename = filename,
	  mimetype = mimetype,
	)
  ), error:
	error "Error uploading file", exc = error.msg
	return RestApiResponse.error(Http500, error.msg)

  codex_api_uploads.inc()
  trace "Uploaded file", cid
  return RestApiResponse.response($cid)
```

> See [Using Codex](https://docs.codex.storage/learn/using) in the Codex docs on how to use the Codex client.

`node.store` (`codex/node.nim`) reads data from the stream, and from each `chunk` it does the following:

```nim
while (let chunk = await chunker.getBytes(); chunk.len > 0):
  without mhash =? MultiHash.digest($hcodec, chunk).mapFailure, err:
    return failure(err)

  without cid =? Cid.init(CIDv1, dataCodec, mhash).mapFailure, err:
    return failure(err)

  without blk =? bt.Block.new(cid, chunk, verify = false):
    return failure("Unable to init block from chunk!")

  cids.add(cid)

  if err =? (await self.networkStore.putBlock(blk)).errorOption:
    error "Unable to store block", cid = blk.cid, err = err.msg
    return failure(&"Unable to store block {blk.cid}")
```

The default chunk size is given by the default block size defined in `codex/codextypes.nim`:

```nim
const
  # Size of blocks for storage / network exchange,
  DefaultBlockSize* = NBytes 1024 * 64
```

### storing blocks

Now, the `networkStore.putBlock`:

```nim
method putBlock*(
    self: NetworkStore, blk: Block, ttl = Duration.none
): Future[?!void] {.async.} =
  ## Store block locally and notify the network
  ##
  let res = await self.localStore.putBlock(blk, ttl)
  if res.isErr:
    return res

  await self.engine.resolveBlocks(@[blk])
  return success()
```

We first store the stuff locally:

```nim
method putBlock*(
    self: RepoStore, blk: Block, ttl = Duration.none
): Future[?!void] {.async.} =
  ## Put a block to the blockstore
  ##

  logScope:
    cid = blk.cid

  let expiry = self.clock.now() + (ttl |? self.blockTtl).seconds

  without res =? await self.storeBlock(blk, expiry), err:
    return failure(err)

  if res.kind == Stored:
    trace "Block Stored"
    if err =? (await self.updateQuotaUsage(plusUsed = res.used)).errorOption:
      # rollback changes
      without delRes =? await self.tryDeleteBlock(blk.cid), err:
        return failure(err)
      return failure(err)

    if err =? (await self.updateTotalBlocksCount(plusCount = 1)).errorOption:
      return failure(err)

    if onBlock =? self.onBlockStored:
      await onBlock(blk.cid)
  else:
    trace "Block already exists"

  return success()
```

The `storeBlock` defined in `codex/stores/repostore/operations.nim` is where we store the block and the metadata.

```nim
proc storeBlock*(
    self: RepoStore, blk: Block, minExpiry: SecondsSince1970
): Future[?!StoreResult] {.async.} =
  if blk.isEmpty:
    return success(StoreResult(kind: AlreadyInStore))

  without metaKey =? createBlockExpirationMetadataKey(blk.cid), err:
    return failure(err)

  without blkKey =? makePrefixKey(self.postFixLen, blk.cid), err:
    return failure(err)

  await self.metaDs.modifyGet(
    ...
  )
```

The `modifyGet` deserve separate treatment.
### modifyGet

`modifyGet` is called on the `metaDs`. `metaDs` has a wrapper type:

```nim
TypedDatastore* = ref object of RootObj
  ds*: Datastore
```

And `ds` above is:

```nim
type
  LevelDbDatastore* = ref object of Datastore
    db: LevelDb
    locks: TableRef[Key, AsyncLock]
```

There is a cascade of callbacks going from `RepoStore` through `TypedDatastore` down to `LevelDbDataStore` as presented on the following sequence diagram:

![[repostore_storeblock.svg]]

> The diagram above can also be viewed [online](https://www.mermaidchart.com/app/projects/2564c095-670f-4258-b8ea-1c5d0b546845/diagrams/b0ba3207-7833-4fd0-b5b1-9d076585d93a/version/v0.1/edit)  (requires [Mermaid Chart](https://www.mermaidchart.com) account)

`LevelDbDataStore` directly interacts with the underlying storage and ensures atomicity of the `modifyGet` operation. `TypedDatastore` performs *encoding* and *decoding* of the data. Finally, `RepoStore` handles metadata creation or update, and also writes the actual block to the underlying block storage via its `repoDS` instance variable.

After the blocks are stored in `repoDS`, back in `node.store` (`CodexNodeRef.store`), we build the Merkle Tree for our block cids and then we compute its root (`treeCid`). Finally, for each block (cid) we compute the [[Codex Merkle Proofs|inclusion proofs]], and we store each `cid`, block `index`, and `proof` under the computed `treeCid`:

```nim
without tree =? CodexTree.init(cids), err:
    return failure(err)

  without treeCid =? tree.rootCid(CIDv1, dataCodec), err:
    return failure(err)

  for index, cid in cids:
    without proof =? tree.getProof(index), err:
      return failure(err)
    if err =?
        (await self.networkStore.putCidAndProof(treeCid, index, cid, proof)).errorOption:
      # TODO add log here
      return failure(err)
```

This concludes the local block storage. We leave the description of `engine.resolveBlocks(@[blk])` for later, when describing the block exchange protocol.

## Downloading content

When we want to download the content from the network, we use `/api/codex/v1/data/{cid}/network/stream` API where we call `await node.retrieveCid(cid.get(), local = false, resp = resp)`.

`node.retrieveCid` tries to get a stream (descendent of libp2p's `LPStream`):

```nim
without stream =? (await node.retrieve(cid, local)), error:
  if error of BlockNotFoundError:
	resp.status = Http404
	return await resp.sendBody("")
  else:
	resp.status = Http500
	return await resp.sendBody(error.msg)
```

This `stream` will be read chunk by chunk (`DefaultBlockSize`) and returned to the client.

To see what the `stream` really will be, we need to dive into  `node.retrieve(cid, local)` (`local` is `false` in this case):

```nim
proc retrieve*(
    self: CodexNodeRef, cid: Cid, local: bool = true
): Future[?!LPStream] {.async.} =
  ## Retrieve by Cid a single block or an entire dataset described by manifest
  ##

  if local and not await (cid in self.networkStore):
    return failure((ref BlockNotFoundError)(msg: "Block not found in local store"))

  without manifest =? (await self.fetchManifest(cid)), err:
    if err of AsyncTimeoutError:
      return failure(err)

    return await self.streamSingleBlock(cid)

  await self.streamEntireDataset(manifest, cid)
```

We first try to get the manifest with `self.fetchManifest(cid)`:

```nim
proc fetchManifest*(self: CodexNodeRef, cid: Cid): Future[?!Manifest] {.async.} =
  ## Fetch and decode a manifest block
  ##

  if err =? cid.isManifest.errorOption:
    return failure "CID has invalid content type for manifest {$cid}"

  trace "Retrieving manifest for cid", cid

  without blk =? await self.networkStore.getBlock(BlockAddress.init(cid)), err:
    trace "Error retrieve manifest block", cid, err = err.msg
    return failure err

  trace "Decoding manifest for cid", cid

  without manifest =? Manifest.decode(blk), err:
    trace "Unable to decode as manifest", err = err.msg
    return failure("Unable to decode as manifest")

  trace "Decoded manifest", cid

  return manifest.success
```

Manifest is ***single block***, and we get it with:

```nim
self.networkStore.getBlock(BlockAddress.init(cid))
```

Here `BlockAddress.init(cid)` reduces to `BlockAddress(leaf: false, cid: cid)`, which means our [[Codex Blocks|BlockAddress]] is just a `Cid`. `getBlock` will try to get the block from the `localStore` first:

```nim
method getBlock*(self: NetworkStore, address: BlockAddress): Future[?!Block] {.async.} =
  without blk =? (await self.localStore.getBlock(address)), err:
    if not (err of BlockNotFoundError):
      error "Error getting block from local store", address, err = err.msg
      return failure err

    without newBlock =? (await self.engine.requestBlock(address)), err:
      error "Unable to get block from exchange engine", address, err = err.msg
      return failure err

    return success newBlock

  return success blk
```

It is `RepoStore`, which by default is set to be a `FSDatastore`: 

```nim
method getBlock*(self: RepoStore, address: BlockAddress): Future[?!Block] =
  ## Get a block from the blockstore
  ##

  if address.leaf:
    self.getBlock(address.treeCid, address.index)
  else:
    self.getBlock(address.cid)
```

Now, we have `leaf` set to `false`, thus we will be using simpler `getBlock` variant:

```nim
method getBlock*(self: RepoStore, cid: Cid): Future[?!Block] {.async.} =
  ## Get a block from the blockstore
  ##

  logScope:
    cid = cid

  if cid.isEmpty:
    trace "Empty block, ignoring"
    return cid.emptyBlock

  without key =? makePrefixKey(self.postFixLen, cid), err:
    trace "Error getting key from provider", err = err.msg
    return failure(err)

  without data =? await self.repoDs.get(key), err:
    if not (err of DatastoreKeyNotFound):
      trace "Error getting block from datastore", err = err.msg, key
      return failure(err)

    return failure(newException(BlockNotFoundError, err.msg))

  trace "Got block for cid", cid
  return Block.new(cid, data, verify = true)
```

If we do not have the block in the `localStore`, we will be trying to get it from the network with `self.engine.requestBlock(address)`:

```nim
proc requestBlock*(
    b: BlockExcEngine, address: BlockAddress
): Future[?!Block] {.async.} =
  let blockFuture = b.pendingBlocks.getWantHandle(address, b.blockFetchTimeout)

  if not b.pendingBlocks.isInFlight(address):
    let peers = b.peers.getPeersForBlock(address)

    if peers.with.len == 0:
      b.discovery.queueFindBlocksReq(@[address.cidOrTreeCid])
    else:
      let selected = pickPseudoRandom(address, peers.with)
      asyncSpawn b.monitorBlockHandle(blockFuture, address, selected.id)
      b.pendingBlocks.setInFlight(address)
      await b.sendWantBlock(@[address], selected)

    await b.sendWantHave(@[address], peers.without)

  # Don't let timeouts bubble up. We can't be too broad here or we break
  # cancellations.
  try:
    success await blockFuture
  except AsyncTimeoutError as err:
    failure err
```

This is also where [[Codex WantList]] topic becomes relevant (and perhaps also [[Codex Block Exchange Protocol]].

After we got the manifest we will proceed with creating a stream through which we will be stream the data down to the browser:

```nim
LPStream(StoreStream.new(self.networkStore, manifest, pad = false)).success
```

The stream abstraction provides a `readOnce` method which will be retrieving the blocks from the `networkStore` and sending the requested bytes down via the stream. `readOnce` is called in `node.retrieveCid`.
