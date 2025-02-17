#codex/upload #codex/download

| related | [[Codex Blocks]], [[Block Storage]], [[Codex Block Exchange Protocol]] |
| ------- | ---------------------------------------------------------------------- |

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

Now, the `netoworkStore.putBlock`:

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

This concludes the local block storage. We leave the description of `engine.resolveBlocks(@[blk])` for later, when describing the block exchange protocol.

## Downloading content

TBD...
