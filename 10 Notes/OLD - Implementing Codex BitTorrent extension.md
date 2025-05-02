In supporting BitTorrent on Codex network, it is important to clarify the pre-conditions: what do we expect to have as an input, and when will be the output.

BitTorrent itself, can have three types of inputs:

- a `.torrent` manifest file - a b-encoded [[BitTorrent metadata files]] - different formats for torrent version one and version 2
- a magnet link - introduced in [[BEP9 - Extension for Peers to Send Metadata Files]] to support trackerless torrent and using DHT for peer discovery

In both cases there are differences between version 1 and version 2 of metadata files (see [[BitTorrent metadata files]] for details) and version 1 and version 2 of [[Magnet Links|magnet links]].

A torrent file, provides a complete description of the torrent, and can be used to compute the corresponding `info` hash.

Thus, while uploading (or seeding) BitTorrent content to the Codex network, the input is the content itself, while the output will be a (hybrid) magnet link.

To retrieve previously seeded content, the input can be a torrent file, a magnet link, or directly an info hash (either v1 or v2, tagged or untagged).

This is illustrated on the following picture:

![[BitTorrent-Upload-Download.svg]]

Thus, from the implementation perspective, the actual input to the Codex network while retrieving previously uploaded content is its `info` hash.

### Uploading BitTorrent Content to Codex

For the time being we only support version 1 and only a single file content (supporting directories and version 2 is work in progress). As a side not, limiting the description to this much simplified version will help to emphasize the important implementation challenges without being distracted with technicalities related to handling multiple file and folders.

Thus, let's assume we have a single input file: `data40k.bin`. It is a binary file of size `40KiB` (`40960` Bytes). We will be using `16KiB` (`16384` Bytes) block size, and commonly used for such small content `piece length` of `256KiB` (`262144` Bytes).

Let's go step by step through the code base to understand the upload process and the related challenges.

First, the upload API:

```
/api/codex/v1/torrent
```

To upload the content we can use the following `POST` request:

```bash
curl -X POST \
  http://localhost:8001/api/codex/v1/torrent \
  -H 'Content-Type: application/octet-stream' \
  -H 'Content-Disposition: filename="data40k.bin"' \
  -w '\n' \
  -T data40k.bin
```

We use `Content Disposition` header to indicate the name we want to use for the uploaded content.

This will land to the API handler in `codex/rest/api.nim` :

```nim
router.rawApi(MethodPost, "/api/codex/v1/torrent") do() -> RestApiResponse:
    ## Upload a file in a streaming manner
    ##
```

It will call `node.storeTorrent` to effectively upload the content and get the resulting `info` (multi) hash back:

```nim
without infoHash =? (
  await node.storeTorrent(
    AsyncStreamWrapper.new(reader = AsyncStreamReader(reader)),
    filename = filename,
    mimetype = mimetype,
  )
), error:
  error "Error uploading file", exc = error.msg
  return RestApiResponse.error(Http500, error.msg)
```

This brings us to `node.storeTorrent` in `codex/node.nim:

```nim
proc storeTorrent*(
    self: CodexNodeRef,
    stream: LPStream,
    filename: ?string = string.none,
    mimetype: ?string = string.none,
): Future[?!MultiHash] {.async.} =
  info "Storing BitTorrent data"

  without bitTorrentManifest =?
    await self.storePieces(
      stream, filename = filename, mimetype = mimetype, blockSize = BitTorrentBlockSize
    ):
    return failure("Unable to store BitTorrent data")

  trace "Created BitTorrent manifest", bitTorrentManifest = $bitTorrentManifest

  let infoBencoded = bencode(bitTorrentManifest.info)

  trace "BitTorrent Info successfully bencoded"

  without infoHash =? MultiHash.digest($Sha1HashCodec, infoBencoded).mapFailure, err:
    return failure(err)

  trace "computed info hash", infoHash = $infoHash

  without manifestBlk =? await self.storeBitTorrentManifest(
    bitTorrentManifest, infoHash
  ), err:
    error "Unable to store manifest"
    return failure(err)

  info "Stored BitTorrent data",
    infoHash = $infoHash, codexManifestCid = bitTorrentManifest.codexManifestCid

  success infoHash
```

It starts with `self.storePieces`, which returns a [[BitTorrent Manifest]]. A manifest contains the BitTorrent Info dictionary and the corresponding Codex Manifest Cid:

```
type
  BitTorrentInfo* = ref object
    length*: uint64
    pieceLength*: uint32
    pieces*: seq[MultiHash]
    name*: ?string

  BitTorrentManifest* = ref object
    info*: BitTorrentInfo
    codexManifestCid*: Cid
```

`storePieces` does a very similar job to the `store` proc which is used for the regular Codex content, but additionally, it computes the *piece hashes* and creates the `info` dictionary and finally returns the corresponding `BitTorrentManifest`.

Back in `storeTorrent`, we *b-encode* the `info` dictionary and compute its hash (multihash). This `info` (multi) hash is what we will use to announce the content on the Codex DHT (see [[Announcing BitTorrent Content on Codex DHT]]).

Finally, `storeBitTorrentManifest` will effectively store the BitTorrent manifest block on the Codex network:

```
proc storeBitTorrentManifest*(
    self: CodexNodeRef, manifest: BitTorrentManifest, infoHash: MultiHash
): Future[?!bt.Block] {.async.} =
  let encodedManifest = manifest.encode()

  without infoHashCid =? Cid.init(CIDv1, InfoHashV1Codec, infoHash).mapFailure, error:
    trace "Unable to create CID for BitTorrent info hash"
    return failure(error)

  without blk =? bt.Block.new(data = encodedManifest, cid = infoHashCid, verify = false),
    error:
    trace "Unable to create block from manifest"
    return failure(error)

  if err =? (await self.networkStore.putBlock(blk)).errorOption:
    trace "Unable to store BitTorrent manifest block", cid = blk.cid, err = err.msg
    return failure(err)

  success blk
```

Some important things happen here. First, notice, that in Codex we use `Cids` to refer to the content. This is very handy: requesting a cid and getting the corresponding data back, we can immediately check if the content multihash present in the Cid, matches the computed multihash of the received data. If they do not match, we know immediately that the received block is invalid.

But looking at the code above, a careful reader will spot immediately that we are *cheating* a bit.

We first create a cid (`infoHashCid`) using precomputed `infoHash`, which we then associate with the `encodedManifest` in the `Block.new` call. Clearly, `info` hash does not identify our `encodedManifest`: if we compute a hash of the `encodedManifest`, it would not match the precomputed `infoHash`. This is because our Bit Torrent Manifest is more than just the `info` dictionary: it also contains the cid of the corresponding Codex Manifest for our content.

This cid is clearly not a valid cid.

We could create a valid Cid, by, for instance, creating a hash over the whole `encodedManifest` and appending it to the precomputed `infoHash` in such a Cid. Then, while retrieving the corresponding block back, we could first compare that the computed hash over the retrieved data matches the hash of the `encodedManifest` that we included in our cid, and then after reconstructing the BitTorrent Manifest from the encoded data, we could b-encode the `info` dictionary from the reconstructed BitTorrent Manifest, compute its hash, and compare it with the precomputed `infoHash` included in the cid. This would make the cid valid, but there is a problem with this approach.

In Codex, we use cids as references to blocks in `RepoStore`. We namely use cids as inputs to functions like `createBlockExpirationMetadataKey` or `makePrefixKey`. The cid itself is not preserved. The uploader (the seeder) has all necessary data to create an extended cid we describe in the paragraph above, but when requesting, the downloader knows only the `info` hash or potentially the contents of the the `.torrent` metadata file. In any case, the downloader does not know the cid of the underlying Codex manifest, pointing to the actual data. This means that the downloader is unable to create a full cid with the appended hash of the full `encodedManifest`. It is technically possible to send such an incomplete cid and use it to retrieve the full cid from the uploader datastore, but we are not making the system any more secure by doing this. The sender, can easily send a forged block with with perfectly valid cid as it has all necessary information to compute the appended hash, but the receiver, not having access to this information beforehand, will not be able to validate it.

Does it mean we can only be sure that the received content identified by the cid of the Codex manifest matches the requested info hash? No.

Notice, that BitTorrent does not use cids. The BitTorrent protocol operates at the level of pieces, and in version 1 of the protocol does not even use inclusion proofs. Yet, it does not wait till the whole piece is fetched in order to conclude it is genuine.

The info dictionary contains the `pieces` attribute, with hashes for all pieces. Once the piece is aggregated from the underlying blocks of `16KiB`, the hash is computed and compared against an entry in the `pieces` array. And this exactly what we do in Codex in order to prove that the received data, identified by the cid of the Codex manifest, matches the requested `info` hash.
Moreover, we also validate the received data at the block level, even before being able to validate the complete piece. We get this as a bonus from the Codex protocol, which together with data block, sends also the corresponding inclusion proof. Thus, even though at the moment we validate the individual blocks, we do not know if the received data, identified by the cid of the Codex manifest, matches the requested `info` hash, we do know already if the received data matches the Codex manifest. If this is not the case, if does not even make sense to aggregate pieces.

Thus, to summarize, while we cannot validate if the received BitTorrent manifest points to the valid data by validating the corresponding cid (`infoHashCid`), we do it later in two phases. Let's look at the download flow, starting from the end.

### Downloading BitTorrent Content from Codex

We start from the `NetworkPeer.readLoop` (in `codex/blockexchange/network/networkpeer.nim`), where we decode the protocol `Message` with:

```nim
data = await conn.readLp(MaxMessageSize.int)
msg = Message.protobufDecode(data).mapFailure().tryGet()
```

There, for each data item, we call:

```nim
BlockDelivery.decode(initProtoBuffer(item, maxSize = MaxBlockSize))
```

and this is where we get the cid, `Block`, `BlockAddress`, and the corresponding `proof` (for regular data, or *leaf* blocks):

```nim
proc decode*(_: type BlockDelivery, pb: ProtoBuffer): ProtoResult[BlockDelivery] =
  var
    value = BlockDelivery()
    dataBuf = newSeq[byte]()
    cidBuf = newSeq[byte]()
    cid: Cid
    ipb: ProtoBuffer

  if ?pb.getField(1, cidBuf):
    cid = ?Cid.init(cidBuf).mapErr(x => ProtoError.IncorrectBlob)
  if ?pb.getField(2, dataBuf):
    value.blk =
      ?Block.new(cid, dataBuf, verify = true).mapErr(x => ProtoError.IncorrectBlob)
  if ?pb.getField(3, ipb):
    value.address = ?BlockAddress.decode(ipb)

  if value.address.leaf:
    var proofBuf = newSeq[byte]()
    if ?pb.getField(4, proofBuf):
      let proof = ?CodexProof.decode(proofBuf).mapErr(x => ProtoError.IncorrectBlob)
      value.proof = proof.some
    else:
      value.proof = CodexProof.none
  else:
    value.proof = CodexProof.none

  ok(value)
```

We see that we while constructing instance of `Block`, we already request the block validation by setting `verify = true`:

```nim
proc new*(
    T: type Block, cid: Cid, data: openArray[byte], verify: bool = true
): ?!Block =
  ## creates a new block for both storage and network IO
  ##

  without isTorrent =? cid.isTorrentCid, err:
    return "Unable to determine if cid is torrent info hash".failure

  # info hash cids are "fake cids" - they will not validate
  # info hash validation is done outside of the cid itself
  if verify and not isTorrent:
    let
      mhash = ?cid.mhash.mapFailure
      computedMhash = ?MultiHash.digest($mhash.mcodec, data).mapFailure
      computedCid = ?Cid.init(cid.cidver, cid.mcodec, computedMhash).mapFailure
    if computedCid != cid:
      return "Cid doesn't match the data".failure

  return Block(cid: cid, data: @data).success
```

Here we see that because as explained above, the cids corresponding to the BitTorrent manifest blocks cannot be immediately validated, we make sure, the validation is skipped here for Torrent cids.

Once the `Message` is decoded, back in `NetworkPeer.readLoop`, it is passed to `NetworkPeer.handler` which is set to `Network.rpcHandler` while creating the instance of `NetworkPeer` in `Network.getOrCreatePeer`. For block deliveries, `Network.rpcHandler` forwards `msg.payload` (`seq[BlockDelivery]`) to `Network.handleBlocksDelivery`, which in turn, calls `Network.handlers.onBlocksDelivery`. The `Network.handlers.onBlocksDelivery` is set by the constructor of `BlockExcEngine`. Thus, in the end of its journey, a `seq[BlockDelivery]` from the `msg.payload` ends up in `BlockExcEngine.blocksDeliveryHandler`. This is where the data blocks are further validated against the inclusion proof and then the validated data (*leafs*) blocks or non-data blocks (non-*leafs*, e.g. a BitTorrent or Codex Manifest block), are stored in the `localStore` and then *resolved* against pending blocks via `BlockExcEngine.resolveBlocks` that calls `pendingBlocks.resolve(blocksDelivery)` (`PendingBlocksManager`). This is where `blockReq.handle.complete(bd.blk)` is called on the matching pending blocks, which completes future awaited in `BlockExcEngine.requestBlock`, which completes the future awaited in `NetworkStore.getBlock`: `await self.engine.requestBlock(address)`. And `NetworkStore.getBlock` was awaited either in `CodexNodeRef.fetchPieces` for data blocks or in `CodexNodeRef.fetchTorrentManifest`.

So, how do we get to `CodexNodeRef.fetchPieces` and `CodexNodeRef.fetchTorrentManifest` in the download flow.

It starts with the API handler of `/api/codex/v1/torrent/{infoHash}/network/stream`:

```nim
router.api(MethodGet, "/api/codex/v1/torrent/{infoHash}/network/stream") do(
    infoHash: MultiHash, resp: HttpResponseRef
  ) -> RestApiResponse:
    var headers = buildCorsHeaders("GET", allowedOrigin)

    without infoHash =? infoHash.mapFailure, error:
      return RestApiResponse.error(Http400, error.msg, headers = headers)

    if infoHash.mcodec != Sha1HashCodec:
      return RestApiResponse.error(
        Http400, "Only torrents version 1 are currently supported!", headers = headers
      )

    if corsOrigin =? allowedOrigin:
      resp.setCorsHeaders("GET", corsOrigin)
      resp.setHeader("Access-Control-Headers", "X-Requested-With")

    trace "torrent requested: ", multihash = $infoHash

    await node.retrieveInfoHash(infoHash, resp = resp)
```

`CodexNodeRef.retrieveInfoHash` first tries to fetch the `Torrent` object, which consists of `torrentManifest` and `codexManifest`. To get it, it calls `node.retrieveTorrent(infoHash)` with the `infoHash` as the argument. And then in the `retrieveTorrent` we get to the above mentioned `fetchTorrentManifest`:

```nim
proc retrieveTorrent*(
    self: CodexNodeRef, infoHash: MultiHash
): Future[?!Torrent] {.async.} =
  without infoHashCid =? Cid.init(CIDv1, InfoHashV1Codec, infoHash).mapFailure, error:
    trace "Unable to create CID for BitTorrent info hash"
    return failure(error)

  without torrentManifest =? (await self.fetchTorrentManifest(infoHashCid)), err:
    trace "Unable to fetch Torrent Manifest"
    return failure(err)

  without codexManifest =? (await self.fetchManifest(torrentManifest.codexManifestCid)),
    err:
    trace "Unable to fetch Codex Manifest for torrent info hash"
    return failure(err)

  success (torrentManifest: torrentManifest, codexManifest: codexManifest)
```

We first create `infoHashCid`, using only the precomputed `infoHash` and we pass it to `fetchTorrentManifest`:

```nim
proc fetchTorrentManifest*(
    self: CodexNodeRef, infoHashCid: Cid
): Future[?!BitTorrentManifest] {.async.} =
  if err =? infoHashCid.isTorrentInfoHash.errorOption:
    return failure "CID has invalid content type for torrent info hash {$cid}"

  trace "Retrieving torrent manifest for infoHashCid", infoHashCid

  without blk =? await self.networkStore.getBlock(BlockAddress.init(infoHashCid)), err:
    trace "Error retrieve manifest block", infoHashCid, err = err.msg
    return failure err

  trace "Successfully retrieved torrent manifest with given block cid",
    cid = blk.cid, infoHashCid
  trace "Decoding torrent manifest"

  without torrentManifest =? BitTorrentManifest.decode(blk), err:
    trace "Unable to decode torrent manifest", err = err.msg
    return failure("Unable to decode torrent manifest")

  trace "Decoded torrent manifest", infoHashCid, torrentManifest = $torrentManifest

  without isValid =? torrentManifest.validate(infoHashCid), err:
    trace "Error validating torrent manifest", infoHashCid, err = err.msg
    return failure(err.msg)

  if not isValid:
    trace "Torrent manifest does not match torrent info hash", infoHashCid
    return failure "Torrent manifest does not match torrent info hash {$infoHashCid}"

  return torrentManifest.success
```

Here we will be awaiting  for the `networkStore.getBlock`, which will get completed with the block delivery flow we describe at the beginning of this section. We restore the `BitTorrentManifest` object using `BitTorrentManifest.decode(blk)`, and then we validate if the `info` dictionary from the received BitTorrent manifest matches the request `infoHash`:

```nim
without isValid =? torrentManifest.validate(infoHashCid), err:
  trace "Error validating torrent manifest", infoHashCid, err = err.msg
  return failure(err.msg)
```

Thus, now we know that we have genuine `info` dictionary.

Now, we still need to get and validate the actual data.

BitTorrent manifest includes the cid of the Codex manifest in `codexManifestCid` attribute. Back in `retrieveTorrent`, we thus now fetch the Codex manifest, and we return both to `retrieveInfoHash`, where the download effectively started.

The `retrieveInfoHash` calls `streamTorrent` passing both manifests as arguments:

```nim
let stream = await node.streamTorrent(torrentManifest, codexManifest)
```

Let's take a look at `streamTorrent`:

```nim
proc streamTorrent*(
    self: CodexNodeRef, torrentManifest: BitTorrentManifest, codexManifest: Manifest
): Future[LPStream] {.async: (raises: []).} =
  trace "Retrieving pieces from torrent"
  let stream = LPStream(StoreStream.new(self.networkStore, codexManifest, pad = false))
  var jobs: seq[Future[void]]

  proc onPieceReceived(blocks: seq[bt.Block], pieceIndex: int): ?!void {.raises: [].} =
    trace "Fetched torrent piece - verifying..."

    var pieceHashCtx: sha1
    pieceHashCtx.init()

    for blk in blocks:
      pieceHashCtx.update(blk.data)

    let pieceHash = pieceHashCtx.finish()

    if (pieceHash != torrentManifest.info.pieces[pieceIndex]):
      error "Piece verification failed", pieceIndex = pieceIndex
      return failure("Piece verification failed")

    trace "Piece verified", pieceIndex, pieceHash
    # great success
    success()

  proc prefetch(): Future[void] {.async: (raises: []).} =
    try:
      if err =? (
        await self.fetchPieces(torrentManifest, codexManifest, onPieceReceived)
      ).errorOption:
        error "Unable to fetch blocks", err = err.msg
        await stream.close()
    except CancelledError:
      trace "Prefetch cancelled"

  jobs.add(prefetch())

  # Monitor stream completion and cancel background jobs when done
  proc monitorStream() {.async: (raises: []).} =
    try:
      await stream.join()
    except CancelledError:
      trace "Stream cancelled"
    finally:
      await noCancel allFutures(jobs.mapIt(it.cancelAndWait))

  self.trackedFutures.track(monitorStream())

  trace "Creating store stream for torrent manifest"
  stream
```

`streamTorrent` does three things:

1. starts background `prefetch` job
2. monitors the stream using `monitorStream`
3. validates the aggregated pieces

The `prefetch` job calls `fetchPieces`:

```nim
proc fetchPieces*(
    self: CodexNodeRef,
    torrentManifest: BitTorrentManifest,
    codexManifest: Manifest,
    onPiece: PieceProc,
): Future[?!void] {.async: (raw: true, raises: [CancelledError]).} =
  trace "Fetching torrent pieces"

  let numOfPieces = torrentManifest.info.pieces.len
  let numOfBlocksPerPiece =
    torrentManifest.info.pieceLength.int div codexManifest.blockSize.int
  let blockIter = Iter[int].new(0 ..< codexManifest.blocksCount)
  let pieceIter = Iter[int].new(0 ..< numOfPieces)
  self.fetchPieces(
    codexManifest.treeCid, blockIter, pieceIter, numOfBlocksPerPiece, onPiece
  )
```

At this level, we create the iterators to manage the sequential processing of blocks and pieces with each piece containing `numOfBlocksPerPiece` blocks. Subsequently, we call the overloaded version of `fetchPieces` that will perform the actual (pre) fetching:

```nim
proc fetchPieces*(
    self: CodexNodeRef,
    cid: Cid,
    blockIter: Iter[int],
    pieceIter: Iter[int],
    numOfBlocksPerPiece: int,
    onPiece: PieceProc,
): Future[?!void] {.async: (raises: [CancelledError]).} =
  while not blockIter.finished:
    let blockFutures = collect:
      for i in 0 ..< numOfBlocksPerPiece:
        if not blockIter.finished:
          let address = BlockAddress.init(cid, blockIter.next())
          self.networkStore.getBlock(address)

    without blocks =? await allFinishedValues(blockFutures), err:
      return failure(err)

    if pieceErr =? (onPiece(blocks, pieceIter.next())).errorOption:
      return failure(pieceErr)

    await sleepAsync(1.millis)

  success()
```

We fetch blocks in batches, or rather per pieces. We trigger fetching blocks with `self.networkStore.getBlock(address)`, which will resolve by either getting the block from the local store or from the network through block delivery described above.

Notice, we need to get all the blocks here, not only trigger fetching the blocks that are not yet available in the local store. This is necessary, because we need to get all the blocks in a piece so that we can validate the piece and potentially stop streaming if the piece turns out to be invalid.

Before calling `onPiece`, where validation will take place, we wait for all `Futures` to complete returning the requested blocks.

`onPiece` is set to `onPieceReceived` in `streamTorrent` and it basically computes the SHA1 hash of the concatenated blocks and checks if it matches the (multi) hash from the `info` dictionary. This steps forms the second validation step: after we checked that the `info` dictionary matches the requested `info` hash in the first step described above, here we are making sure that the received content matches the metadata in the `info` dictionary, and thus it is indeed the content identified by the `info` hash from the request.

`fetchPieces` operates in background, and thus very likely after first piece has been fetched the stream will be returned to `retrieveInfoHash` where streaming the blocks down to the client will take place:

```nim
proc retrieveInfoHash(
    node: CodexNodeRef, infoHash: MultiHash, resp: HttpResponseRef
): Future[void] {.async.} =
  ## Download torrent from the node in a streaming
  ## manner
  ##
  var stream: LPStream

  var bytes = 0
  try:
    without torrent =? (await node.retrieveTorrent(infoHash)), err:
      error "Unable to fetch Torrent Metadata", err = err.msg
      resp.status = Http404
      await resp.sendBody(err.msg)
      return
    let (torrentManifest, codexManifest) = torrent

    if codexManifest.mimetype.isSome:
      resp.setHeader("Content-Type", codexManifest.mimetype.get())
    else:
      resp.addHeader("Content-Type", "application/octet-stream")

    if codexManifest.filename.isSome:
      resp.setHeader(
        "Content-Disposition",
        "attachment; filename=\"" & codexManifest.filename.get() & "\"",
      )
    else:
      resp.setHeader("Content-Disposition", "attachment")

    await resp.prepareChunked()

    let stream = await node.streamTorrent(torrentManifest, codexManifest)

    while not stream.atEof:
      var
        buff = newSeqUninitialized[byte](BitTorrentBlockSize.int)
        len = await stream.readOnce(addr buff[0], buff.len)

      buff.setLen(len)
      if buff.len <= 0:
        break

      bytes += buff.len

      await resp.sendChunk(addr buff[0], buff.len)
    await resp.finish()
    codex_api_downloads.inc()
  except CancelledError as exc:
    raise exc
  except CatchableError as exc:
    warn "Error streaming blocks", exc = exc.msg
    resp.status = Http500
    if resp.isPending():
      await resp.sendBody(exc.msg)
  finally:
    info "Sent bytes for torrent", infoHash = $infoHash, bytes
    if not stream.isNil:
      await stream.close()
```

Now, two important points. First, when the streaming happens to be interrupted the stream will be closed in the `finally` block. This in turns will be detected by the `monitorStream` in `streamTorrent` causing the `prefetch` job to be cancelled. Second, when either piece validation fails, or if any of the `getBlock` future awaiting completion fails, `prefetch` will return error, which will cause the stream to be closed:

```nim
proc prefetch(): Future[void] {.async: (raises: []).} =
  try:
    if err =? (
	  await self.fetchPieces(torrentManifest, codexManifest, onPieceReceived)
    ).errorOption:
	  error "Unable to fetch blocks", err = err.msg
	  await stream.close()
   except CancelledError:
   trace "Prefetch cancelled"
```

Without this detection mechanism, we would either continue fetching blocks even when streaming API request has been interrupted, or we would continue streaming, even when it is already known that the piece validation phase has failed.  This would result in invalid content being returned to the client. After any failure in the `prefetch` job, the pieces will no longer be validated, thus it does not make any sense to continue the streaming operation, which otherwise, would cause fetching blocks one-by-one in the streaming loop in `retrieve`:

```nim
while not stream.atEof:
  var
	buff = newSeqUninitialized[byte](BitTorrentBlockSize.int)
	len = await stream.readOnce(addr buff[0], buff.len)

  buff.setLen(len)
  if buff.len <= 0:
	break

  bytes += buff.len

  await resp.sendChunk(addr buff[0], buff.len)
```

The `stream.readOnce` implemented in `StoreStream`, which uses the same underlying `networkStore` that is also used in `fetchPieces` proc shown above, will be calling that same `getBlock` operation, which in case the block is not already in local store (because it was already there or as a result of the prefetch operation), will request it from the block exchange engine via `BlockExcEngine.requestBlock` operation. In case there is already a pending request for the given block address, the `PendingBlocksManager` will return the existing block handle, so that  `BlockExcEngine.requestBlock` operation will not cause duplicate request. It will, however potentially return an invalid block to the client, before the containing piece has been validated in the prefetch phase. 

> [!danger]
This may occasionally cause an unlikely event in which both `fetchPieces` in the `prefetch` job and `readOnce` on the `StoreStream` will be awaiting on the `getBlock` of the very last block request, and `readOnce` will regain the control before `fetchPieces` does. In such a case, the client may complete the streaming operation successfully, even if the corresponding last piece validation fails.
> 
> OR, in even more unlikely event that all the blocks belonging to the last piece are in the local store, `readOnce` may consume them before the last piece is validated.
> 
> We should perhaps make sure that the very last piece can only be streamed to the client after the `prefetch` operation completes.
> 

```bash
TRC 2025-03-19 16:11:06.334+01:00 torrent requested:                         topics="codex restapi" tid=5050038 multihash=sha1/4249FFB943675890CF09342629CD3782D107B709
TRC 2025-03-19 16:11:06.334+01:00 Retrieving torrent manifest for infoHashCid topics="codex node" tid=5050038 infoHashCid=z8q*fCdDsv
TRC 2025-03-19 16:11:06.340+01:00 Successfully retrieved torrent manifest with given block cid topics="codex node" tid=5050038 cid=z8q*fCdDsv infoHashCid=z8q*fCdDsv
TRC 2025-03-19 16:11:06.340+01:00 Decoding torrent manifest                  topics="codex node" tid=5050038
TRC 2025-03-19 16:11:06.340+01:00 Decoded torrent manifest                   topics="codex node" tid=5050038 infoHashCid=z8q*fCdDsv torrentManifest="BitTorrentManifest(info: BitTorrentInfo(length: 10485760, pieceLength: 262144, pieces: @[sha1/5B3B77A971432C07B1C3C36A73BA5E79DBAE20EE, sha1/322890271DAE134970AABAA9CB30AF7464971599, sha1/2394F90BABA3677D44DDA0565543591AC26CA584, sha1/36EF17CDEFE621AE433A0A30EF3F6A20E0D07DB2, sha1/C21F2C1A1A6BF6E3E6CA75C13969F32EBE802867, sha1/B576B2118EC0B5B0D6AC5BB60BF12F881E5E395C, sha1/3FC4F41C6A83C8411BAE41E83AE327299FA31A98, sha1/92BD07E01D40E8BFB7C3B4597568B44F805508B5, sha1/DEC4E1492D2447436AF54AF3FC6319D409E2F48F, sha1/3E4BFDB4F970A9EDA9906E007073B0FB94A4F91A, sha1/CFB9B01285FA00BC43C5E439A65958D8CBBCD046, sha1/EAFB36F2DD5AA8EE80811946C26DFB8F95ABB18A, sha1/FEDFD7A30E09B395D8F78148CB826127070272DD, sha1/F1588A342C11776DB7DC8C5BCFA72A1701DF0442, sha1/34D4E095D53A0CA33375444A19652BD866027350, sha1/AE7FF2CA95BF751B68D2E22BB015435347CFE49C, sha1/85FCC1B3A8CE7D5C0397E8D27084C067A3067496, sha1/88C3DF9C35A23FE6B1E363F606068F15DB5EE093, sha1/98F7A4A6113A3CC9CB3EF086A2B9E36DAA92B78D, sha1/91DDF5B1F25715C17CD709019B71D2233A142CC1, sha1/1A850C78AB1CB596D8298371132A135DB96D5943, sha1/88E8F31AE70A6A81E25FF9E6D2DC824F07F1AF9A, sha1/A335A0DE3F7E1F4191D89E68F9AE9D4406CE4449, sha1/7AC5488A3B6C8A93F7DF3DE64BBCF2FA1F185F4D, sha1/7A88908AF090C1F8FC2BCF51C8BBB6F92D13AE01, sha1/BBEBBD427BEE80178C291366A570A355204E67CA, sha1/0112EDE76962115D7DDD24F7D14BE41BCCD51634, sha1/EFBEFC1DFBB0F7676447C9D0936775A8933008D2, sha1/6508DE0FD7FC8D8EE300C88F0743A52CD79CB616, sha1/F1B02EFB043CC6F37FE22C59055960AAC4ABDCC6, sha1/74FDA7F48FC089AA7A684D4E7C2C1F4BC5B08980, sha1/FAFAC020F2C1DEC260581810F8BB27BBCB211AFD, sha1/1AB425BA8DD6A2AD1C469417C7F7CB54E3226B82, sha1/08338248DAA53ADCBF65D052E93EB8BC677E14B3, sha1/BAD3B38D731F16B3F6663131EEE06E56EBFAB7C2, sha1/8651B73B110025390F4DDCE6AFD43F34ED65A0BE, sha1/486DCD80D8F25432E31BF59C8818003DFE8709F0, sha1/336038716957C64D3124EA1985D8180EE6A097E8, sha1/C7E2B6ADCAB60B57A9A47CCE8454BDAE46E6EE57, sha1/8B6DA36E26C2654B6407253A7816343F818DEDE5], name: some(\"data10M.bin\")), codexManifestCid: zDvZRwzm1GAigGvTcrXyC2cATBL9W2xNPqjKLKb835pHu4Xcm1E6)"
TRC 2025-03-19 16:11:06.341+01:00 Retrieving manifest for cid                topics="codex node" tid=5050038 cid=zDv*Xcm1E6
TRC 2025-03-19 16:11:06.343+01:00 Decoding manifest for cid                  topics="codex node" tid=5050038 cid=zDv*Xcm1E6
TRC 2025-03-19 16:11:06.343+01:00 Decoded manifest                           topics="codex node" tid=5050038 cid=zDv*Xcm1E6
TRC 2025-03-19 16:11:06.344+01:00 Retrieving pieces from torrent             topics="codex node" tid=5050038
TRC 2025-03-19 16:11:06.344+01:00 Fetching torrent pieces                    topics="codex node" tid=5050038
TRC 2025-03-19 16:11:06.344+01:00 Creating store stream for torrent manifest topics="codex node" tid=5050038
TRC 2025-03-19 16:11:06.344+01:00 Waiting for piece                          topics="codex restapi" tid=5050038 pieceIndex=0
TRC 2025-03-19 16:11:06.371+01:00 Fetched torrent piece - verifying...       topics="codex node" tid=5050038
TRC 2025-03-19 16:11:06.372+01:00 Piece verified                             topics="codex node" tid=5050038 pieceIndex=0
TRC 2025-03-19 16:11:06.372+01:00 Got piece                                  topics="codex restapi" tid=5050038 pieceIndex=0
TRC 2025-03-19 16:11:06.385+01:00 Waiting for piece                          topics="codex restapi" tid=5050038 pieceIndex=1
TRC 2025-03-19 16:11:06.411+01:00 Fetched torrent piece - verifying...       topics="codex node" tid=5050038
TRC 2025-03-19 16:11:06.412+01:00 Piece verified                             topics="codex node" tid=5050038 pieceIndex=1
TRC 2025-03-19 16:11:06.412+01:00 Got piece                                  topics="codex restapi" tid=5050038 pieceIndex=1
TRC 2025-03-19 16:11:06.419+01:00 Waiting for piece                          topics="codex restapi" tid=5050038 pieceIndex=2
TRC 2025-03-19 16:11:06.431+01:00 Fetched torrent piece - verifying...       topics="codex node" tid=5050038
TRC 2025-03-19 16:11:06.431+01:00 Piece verified                             topics="codex node" tid=5050038 pieceIndex=2
TRC 2025-03-19 16:11:06.431+01:00 Got piece                                  topics="codex restapi" tid=5050038 pieceIndex=2
TRC 2025-03-19 16:11:06.438+01:00 Waiting for piece                          topics="codex restapi" tid=5050038 pieceIndex=3
TRC 2025-03-19 16:11:06.471+01:00 Fetched torrent piece - verifying...       topics="codex node" tid=5050038
TRC 2025-03-19 16:11:06.471+01:00 Piece verified                             topics="codex node" tid=5050038 pieceIndex=3
TRC 2025-03-19 16:11:06.471+01:00 Got piece                                  topics="codex restapi" tid=5050038 pieceIndex=3
TRC 2025-03-19 16:11:06.480+01:00 Waiting for piece                          topics="codex restapi" tid=5050038 pieceIndex=4
TRC 2025-03-19 16:11:06.490+01:00 Fetched torrent piece - verifying...       topics="codex node" tid=5050038
TRC 2025-03-19 16:11:06.491+01:00 Piece verified                             topics="codex node" tid=5050038 pieceIndex=4
TRC 2025-03-19 16:11:06.491+01:00 Got piece                                  topics="codex restapi" tid=5050038 pieceIndex=4
TRC 2025-03-19 16:11:06.496+01:00 Waiting for piece                          topics="codex restapi" tid=5050038 pieceIndex=5
TRC 2025-03-19 16:11:06.524+01:00 Fetched torrent piece - verifying...       topics="codex node" tid=5050038
TRC 2025-03-19 16:11:06.524+01:00 Piece verified                             topics="codex node" tid=5050038 pieceIndex=5
TRC 2025-03-19 16:11:06.524+01:00 Got piece                                  topics="codex restapi" tid=5050038 pieceIndex=5
TRC 2025-03-19 16:11:06.531+01:00 Waiting for piece                          topics="codex restapi" tid=5050038 pieceIndex=6
TRC 2025-03-19 16:11:06.549+01:00 Fetched torrent piece - verifying...       topics="codex node" tid=5050038
TRC 2025-03-19 16:11:06.549+01:00 Piece verified                             topics="codex node" tid=5050038 pieceIndex=6
TRC 2025-03-19 16:11:06.549+01:00 Got piece                                  topics="codex restapi" tid=5050038 pieceIndex=6
TRC 2025-03-19 16:11:06.556+01:00 Waiting for piece                          topics="codex restapi" tid=5050038 pieceIndex=7
TRC 2025-03-19 16:11:06.575+01:00 Fetched torrent piece - verifying...       topics="codex node" tid=5050038
TRC 2025-03-19 16:11:06.575+01:00 Piece verified                             topics="codex node" tid=5050038 pieceIndex=7
TRC 2025-03-19 16:11:06.575+01:00 Got piece                                  topics="codex restapi" tid=5050038 pieceIndex=7
TRC 2025-03-19 16:11:06.583+01:00 Waiting for piece                          topics="codex restapi" tid=5050038 pieceIndex=8
TRC 2025-03-19 16:11:06.599+01:00 Fetched torrent piece - verifying...       topics="codex node" tid=5050038
TRC 2025-03-19 16:11:06.600+01:00 Piece verified                             topics="codex node" tid=5050038 pieceIndex=8
TRC 2025-03-19 16:11:06.600+01:00 Got piece                                  topics="codex restapi" tid=5050038 pieceIndex=8
TRC 2025-03-19 16:11:06.605+01:00 Waiting for piece                          topics="codex restapi" tid=5050038 pieceIndex=9
TRC 2025-03-19 16:11:06.624+01:00 Fetched torrent piece - verifying...       topics="codex node" tid=5050038
TRC 2025-03-19 16:11:06.625+01:00 Piece verified                             topics="codex node" tid=5050038 pieceIndex=9
TRC 2025-03-19 16:11:06.625+01:00 Got piece                                  topics="codex restapi" tid=5050038 pieceIndex=9
TRC 2025-03-19 16:11:06.631+01:00 Waiting for piece                          topics="codex restapi" tid=5050038 pieceIndex=10
TRC 2025-03-19 16:11:06.643+01:00 Fetched torrent piece - verifying...       topics="codex node" tid=5050038
TRC 2025-03-19 16:11:06.643+01:00 Piece verified                             topics="codex node" tid=5050038 pieceIndex=10
TRC 2025-03-19 16:11:06.643+01:00 Got piece                                  topics="codex restapi" tid=5050038 pieceIndex=10
TRC 2025-03-19 16:11:06.648+01:00 Waiting for piece                          topics="codex restapi" tid=5050038 pieceIndex=11
TRC 2025-03-19 16:11:06.675+01:00 Fetched torrent piece - verifying...       topics="codex node" tid=5050038
TRC 2025-03-19 16:11:06.675+01:00 Piece verified                             topics="codex node" tid=5050038 pieceIndex=11
TRC 2025-03-19 16:11:06.675+01:00 Got piece                                  topics="codex restapi" tid=5050038 pieceIndex=11
TRC 2025-03-19 16:11:06.682+01:00 Waiting for piece                          topics="codex restapi" tid=5050038 pieceIndex=12
TRC 2025-03-19 16:11:06.693+01:00 Fetched torrent piece - verifying...       topics="codex node" tid=5050038
TRC 2025-03-19 16:11:06.694+01:00 Piece verified                             topics="codex node" tid=5050038 pieceIndex=12
TRC 2025-03-19 16:11:06.694+01:00 Got piece                                  topics="codex restapi" tid=5050038 pieceIndex=12
TRC 2025-03-19 16:11:06.699+01:00 Waiting for piece                          topics="codex restapi" tid=5050038 pieceIndex=13
TRC 2025-03-19 16:11:06.725+01:00 Fetched torrent piece - verifying...       topics="codex node" tid=5050038
TRC 2025-03-19 16:11:06.725+01:00 Piece verified                             topics="codex node" tid=5050038 pieceIndex=13
TRC 2025-03-19 16:11:06.726+01:00 Got piece                                  topics="codex restapi" tid=5050038 pieceIndex=13
TRC 2025-03-19 16:11:06.732+01:00 Waiting for piece                          topics="codex restapi" tid=5050038 pieceIndex=14
TRC 2025-03-19 16:11:06.744+01:00 Fetched torrent piece - verifying...       topics="codex node" tid=5050038
TRC 2025-03-19 16:11:06.744+01:00 Piece verified                             topics="codex node" tid=5050038 pieceIndex=14
TRC 2025-03-19 16:11:06.744+01:00 Got piece                                  topics="codex restapi" tid=5050038 pieceIndex=14
TRC 2025-03-19 16:11:06.749+01:00 Waiting for piece                          topics="codex restapi" tid=5050038 pieceIndex=15
TRC 2025-03-19 16:11:06.775+01:00 Fetched torrent piece - verifying...       topics="codex node" tid=5050038
TRC 2025-03-19 16:11:06.776+01:00 Piece verified                             topics="codex node" tid=5050038 pieceIndex=15
TRC 2025-03-19 16:11:06.776+01:00 Got piece                                  topics="codex restapi" tid=5050038 pieceIndex=15
TRC 2025-03-19 16:11:06.783+01:00 Waiting for piece                          topics="codex restapi" tid=5050038 pieceIndex=16
TRC 2025-03-19 16:11:06.794+01:00 Fetched torrent piece - verifying...       topics="codex node" tid=5050038
TRC 2025-03-19 16:11:06.794+01:00 Piece verified                             topics="codex node" tid=5050038 pieceIndex=16
TRC 2025-03-19 16:11:06.794+01:00 Got piece                                  topics="codex restapi" tid=5050038 pieceIndex=16
TRC 2025-03-19 16:11:06.800+01:00 Waiting for piece                          topics="codex restapi" tid=5050038 pieceIndex=17
TRC 2025-03-19 16:11:06.825+01:00 Fetched torrent piece - verifying...       topics="codex node" tid=5050038
TRC 2025-03-19 16:11:06.826+01:00 Piece verified                             topics="codex node" tid=5050038 pieceIndex=17
TRC 2025-03-19 16:11:06.826+01:00 Got piece                                  topics="codex restapi" tid=5050038 pieceIndex=17
TRC 2025-03-19 16:11:06.832+01:00 Waiting for piece                          topics="codex restapi" tid=5050038 pieceIndex=18
TRC 2025-03-19 16:11:06.844+01:00 Fetched torrent piece - verifying...       topics="codex node" tid=5050038
TRC 2025-03-19 16:11:06.844+01:00 Piece verified                             topics="codex node" tid=5050038 pieceIndex=18
TRC 2025-03-19 16:11:06.844+01:00 Got piece                                  topics="codex restapi" tid=5050038 pieceIndex=18
TRC 2025-03-19 16:11:06.850+01:00 Waiting for piece                          topics="codex restapi" tid=5050038 pieceIndex=19
TRC 2025-03-19 16:11:06.877+01:00 Fetched torrent piece - verifying...       topics="codex node" tid=5050038
TRC 2025-03-19 16:11:06.877+01:00 Piece verified                             topics="codex node" tid=5050038 pieceIndex=19
TRC 2025-03-19 16:11:06.877+01:00 Got piece                                  topics="codex restapi" tid=5050038 pieceIndex=19
TRC 2025-03-19 16:11:06.884+01:00 Waiting for piece                          topics="codex restapi" tid=5050038 pieceIndex=20
TRC 2025-03-19 16:11:06.902+01:00 Fetched torrent piece - verifying...       topics="codex node" tid=5050038
TRC 2025-03-19 16:11:06.902+01:00 Piece verified                             topics="codex node" tid=5050038 pieceIndex=20
TRC 2025-03-19 16:11:06.902+01:00 Got piece                                  topics="codex restapi" tid=5050038 pieceIndex=20
TRC 2025-03-19 16:11:06.907+01:00 Waiting for piece                          topics="codex restapi" tid=5050038 pieceIndex=21
TRC 2025-03-19 16:11:06.926+01:00 Fetched torrent piece - verifying...       topics="codex node" tid=5050038
TRC 2025-03-19 16:11:06.927+01:00 Piece verified                             topics="codex node" tid=5050038 pieceIndex=21
TRC 2025-03-19 16:11:06.927+01:00 Got piece                                  topics="codex restapi" tid=5050038 pieceIndex=21
TRC 2025-03-19 16:11:06.933+01:00 Waiting for piece                          topics="codex restapi" tid=5050038 pieceIndex=22
TRC 2025-03-19 16:11:06.945+01:00 Fetched torrent piece - verifying...       topics="codex node" tid=5050038
TRC 2025-03-19 16:11:06.945+01:00 Piece verified                             topics="codex node" tid=5050038 pieceIndex=22
TRC 2025-03-19 16:11:06.945+01:00 Got piece                                  topics="codex restapi" tid=5050038 pieceIndex=22
TRC 2025-03-19 16:11:06.951+01:00 Waiting for piece                          topics="codex restapi" tid=5050038 pieceIndex=23
TRC 2025-03-19 16:11:06.978+01:00 Fetched torrent piece - verifying...       topics="codex node" tid=5050038
TRC 2025-03-19 16:11:06.979+01:00 Piece verified                             topics="codex node" tid=5050038 pieceIndex=23
TRC 2025-03-19 16:11:06.980+01:00 Got piece                                  topics="codex restapi" tid=5050038 pieceIndex=23
TRC 2025-03-19 16:11:06.994+01:00 Waiting for piece                          topics="codex restapi" tid=5050038 pieceIndex=24
TRC 2025-03-19 16:11:07.023+01:00 Fetched torrent piece - verifying...       topics="codex node" tid=5050038
TRC 2025-03-19 16:11:07.023+01:00 Piece verified                             topics="codex node" tid=5050038 pieceIndex=24
TRC 2025-03-19 16:11:07.023+01:00 Got piece                                  topics="codex restapi" tid=5050038 pieceIndex=24
TRC 2025-03-19 16:11:07.033+01:00 Waiting for piece                          topics="codex restapi" tid=5050038 pieceIndex=25
TRC 2025-03-19 16:11:07.049+01:00 Fetched torrent piece - verifying...       topics="codex node" tid=5050038
TRC 2025-03-19 16:11:07.049+01:00 Piece verified                             topics="codex node" tid=5050038 pieceIndex=25
TRC 2025-03-19 16:11:07.049+01:00 Got piece                                  topics="codex restapi" tid=5050038 pieceIndex=25
TRC 2025-03-19 16:11:07.055+01:00 Waiting for piece                          topics="codex restapi" tid=5050038 pieceIndex=26
TRC 2025-03-19 16:11:07.084+01:00 Fetched torrent piece - verifying...       topics="codex node" tid=5050038
TRC 2025-03-19 16:11:07.084+01:00 Piece verified                             topics="codex node" tid=5050038 pieceIndex=26
TRC 2025-03-19 16:11:07.084+01:00 Got piece                                  topics="codex restapi" tid=5050038 pieceIndex=26
TRC 2025-03-19 16:11:07.091+01:00 Waiting for piece                          topics="codex restapi" tid=5050038 pieceIndex=27
TRC 2025-03-19 16:11:07.121+01:00 Fetched torrent piece - verifying...       topics="codex node" tid=5050038
TRC 2025-03-19 16:11:07.121+01:00 Piece verified                             topics="codex node" tid=5050038 pieceIndex=27
TRC 2025-03-19 16:11:07.121+01:00 Got piece                                  topics="codex restapi" tid=5050038 pieceIndex=27
TRC 2025-03-19 16:11:07.128+01:00 Waiting for piece                          topics="codex restapi" tid=5050038 pieceIndex=28
TRC 2025-03-19 16:11:07.140+01:00 Fetched torrent piece - verifying...       topics="codex node" tid=5050038
TRC 2025-03-19 16:11:07.140+01:00 Piece verified                             topics="codex node" tid=5050038 pieceIndex=28
TRC 2025-03-19 16:11:07.141+01:00 Got piece                                  topics="codex restapi" tid=5050038 pieceIndex=28
TRC 2025-03-19 16:11:07.148+01:00 Waiting for piece                          topics="codex restapi" tid=5050038 pieceIndex=29
TRC 2025-03-19 16:11:07.401+01:00 Fetched torrent piece - verifying...       topics="codex node" tid=5050038
TRC 2025-03-19 16:11:07.401+01:00 Piece verified                             topics="codex node" tid=5050038 pieceIndex=29
TRC 2025-03-19 16:11:07.401+01:00 Got piece                                  topics="codex restapi" tid=5050038 pieceIndex=29
TRC 2025-03-19 16:11:07.409+01:00 Waiting for piece                          topics="codex restapi" tid=5050038 pieceIndex=30
TRC 2025-03-19 16:11:07.442+01:00 Fetched torrent piece - verifying...       topics="codex node" tid=5050038
TRC 2025-03-19 16:11:07.442+01:00 Piece verified                             topics="codex node" tid=5050038 pieceIndex=30
TRC 2025-03-19 16:11:07.442+01:00 Got piece                                  topics="codex restapi" tid=5050038 pieceIndex=30
TRC 2025-03-19 16:11:07.449+01:00 Waiting for piece                          topics="codex restapi" tid=5050038 pieceIndex=31
TRC 2025-03-19 16:11:07.481+01:00 Fetched torrent piece - verifying...       topics="codex node" tid=5050038
TRC 2025-03-19 16:11:07.481+01:00 Piece verified                             topics="codex node" tid=5050038 pieceIndex=31
TRC 2025-03-19 16:11:07.481+01:00 Got piece                                  topics="codex restapi" tid=5050038 pieceIndex=31
TRC 2025-03-19 16:11:07.488+01:00 Waiting for piece                          topics="codex restapi" tid=5050038 pieceIndex=32
TRC 2025-03-19 16:11:07.499+01:00 Fetched torrent piece - verifying...       topics="codex node" tid=5050038
TRC 2025-03-19 16:11:07.500+01:00 Piece verified                             topics="codex node" tid=5050038 pieceIndex=32
TRC 2025-03-19 16:11:07.500+01:00 Got piece                                  topics="codex restapi" tid=5050038 pieceIndex=32
TRC 2025-03-19 16:11:07.505+01:00 Waiting for piece                          topics="codex restapi" tid=5050038 pieceIndex=33
TRC 2025-03-19 16:11:07.536+01:00 Fetched torrent piece - verifying...       topics="codex node" tid=5050038
TRC 2025-03-19 16:11:07.536+01:00 Piece verified                             topics="codex node" tid=5050038 pieceIndex=33
TRC 2025-03-19 16:11:07.536+01:00 Got piece                                  topics="codex restapi" tid=5050038 pieceIndex=33
TRC 2025-03-19 16:11:07.544+01:00 Waiting for piece                          topics="codex restapi" tid=5050038 pieceIndex=34
TRC 2025-03-19 16:11:07.555+01:00 Fetched torrent piece - verifying...       topics="codex node" tid=5050038
TRC 2025-03-19 16:11:07.556+01:00 Piece verified                             topics="codex node" tid=5050038 pieceIndex=34
TRC 2025-03-19 16:11:07.556+01:00 Got piece                                  topics="codex restapi" tid=5050038 pieceIndex=34
TRC 2025-03-19 16:11:07.561+01:00 Waiting for piece                          topics="codex restapi" tid=5050038 pieceIndex=35
TRC 2025-03-19 16:11:07.587+01:00 Fetched torrent piece - verifying...       topics="codex node" tid=5050038
TRC 2025-03-19 16:11:07.588+01:00 Piece verified                             topics="codex node" tid=5050038 pieceIndex=35
TRC 2025-03-19 16:11:07.588+01:00 Got piece                                  topics="codex restapi" tid=5050038 pieceIndex=35
TRC 2025-03-19 16:11:07.594+01:00 Waiting for piece                          topics="codex restapi" tid=5050038 pieceIndex=36
TRC 2025-03-19 16:11:07.606+01:00 Fetched torrent piece - verifying...       topics="codex node" tid=5050038
TRC 2025-03-19 16:11:07.606+01:00 Piece verified                             topics="codex node" tid=5050038 pieceIndex=36
TRC 2025-03-19 16:11:07.606+01:00 Got piece                                  topics="codex restapi" tid=5050038 pieceIndex=36
TRC 2025-03-19 16:11:07.612+01:00 Waiting for piece                          topics="codex restapi" tid=5050038 pieceIndex=37
TRC 2025-03-19 16:11:07.637+01:00 Fetched torrent piece - verifying...       topics="codex node" tid=5050038
TRC 2025-03-19 16:11:07.638+01:00 Piece verified                             topics="codex node" tid=5050038 pieceIndex=37
TRC 2025-03-19 16:11:07.638+01:00 Got piece                                  topics="codex restapi" tid=5050038 pieceIndex=37
TRC 2025-03-19 16:11:07.646+01:00 Waiting for piece                          topics="codex restapi" tid=5050038 pieceIndex=38
TRC 2025-03-19 16:11:07.663+01:00 Fetched torrent piece - verifying...       topics="codex node" tid=5050038
TRC 2025-03-19 16:11:07.663+01:00 Piece verified                             topics="codex node" tid=5050038 pieceIndex=38
TRC 2025-03-19 16:11:07.663+01:00 Got piece                                  topics="codex restapi" tid=5050038 pieceIndex=38
TRC 2025-03-19 16:11:07.669+01:00 Waiting for piece                          topics="codex restapi" tid=5050038 pieceIndex=39
TRC 2025-03-19 16:11:07.691+01:00 Fetched torrent piece - verifying...       topics="codex node" tid=5050038
TRC 2025-03-19 16:11:07.692+01:00 Piece verified                             topics="codex node" tid=5050038 pieceIndex=39
TRC 2025-03-19 16:11:07.692+01:00 Got piece                                  topics="codex restapi" tid=5050038 pieceIndex=39
INF 2025-03-19 16:11:07.696+01:00 Sent bytes for torrent                     topics="codex restapi" tid=5050038 infoHash=sha1/4249FFB943675890CF09342629CD3782D107B709 bytes=10485760

```