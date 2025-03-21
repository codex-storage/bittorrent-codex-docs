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

With this we know that we have a genuine `info` dictionary.

Now, we still need to get and validate the actual data.

BitTorrent manifest includes the cid of the Codex manifest in `codexManifestCid` attribute. Back in `retrieveTorrent`, we now fetch the Codex manifest, and we return both to `retrieveInfoHash`, where the download effectively started.

We are ready to start the streaming operation. The illustration below, shows a high level conceptual overview of the streaming process:

![[Codex BitTorrent Streaming.svg]]

From the diagram above we see, that there are two concurrent tasks: a *prefetch* tasks fetching the blocks from the network store, aggregating, and validating pieces, and a *streaming* tasks, sending the blocks down to the client via REST API.

The prefetch task is started by `retrieveInfoHash` calling `streamTorrent`, passing both manifests and piece validator as arguments:

```nim
let torrentPieceValidator = newTorrentPieceValidator(torrentManifest, codexManifest)

let stream =
  await node.streamTorrent(torrentManifest, codexManifest, torrentPieceValidator)
```

Let's take a look at `streamTorrent`:

```nim
proc streamTorrent*(
    self: CodexNodeRef,
    torrentManifest: BitTorrentManifest,
    codexManifest: Manifest,
    pieceValidator: TorrentPieceValidator,
): Future[LPStream] {.async: (raises: []).} =
  trace "Retrieving pieces from torrent"
  let stream = LPStream(StoreStream.new(self.networkStore, codexManifest, pad = false))

  proc prefetch(): Future[void] {.async: (raises: []).} =
    try:
      if err =? (await self.fetchPieces(torrentManifest, codexManifest, pieceValidator)).errorOption:
        error "Unable to fetch blocks", err = err.msg
        await noCancel pieceValidator.cancel()
        await noCancel stream.close()
    except CancelledError:
      trace "Prefetch cancelled"

  let prefetchTask = prefetch()

  # Monitor stream completion and cancel background jobs when done
  proc monitorStream() {.async: (raises: []).} =
    try:
      await stream.join()
    except CancelledError:
      trace "Stream cancelled"
    finally:
      await noCancel prefetchTask.cancelAndWait

  self.trackedFutures.track(monitorStream())

  trace "Creating store stream for torrent manifest"
  stream
```

`streamTorrent` does two things:

1. starts background `prefetch` task
2. monitors the stream using `monitorStream`

The `prefetch` job calls `fetchPieces`:

```nim
proc fetchPieces*(
    self: CodexNodeRef,
    torrentManifest: BitTorrentManifest,
    codexManifest: Manifest,
    pieceValidator: TorrentPieceValidator,
): Future[?!void] {.async: (raises: [CancelledError]).} =
  let cid = codexManifest.treeCid
  let numOfBlocksPerPiece = pieceValidator.numberOfBlocksPerPiece
  let blockIter = Iter[int].new(0 ..< codexManifest.blocksCount)
  while not blockIter.finished:
    let blockFutures = collect:
      for i in 0 ..< numOfBlocksPerPiece:
        if not blockIter.finished:
          let address = BlockAddress.init(cid, blockIter.next())
          self.networkStore.getBlock(address)

    without blocks =? await allFinishedValues(blockFutures), err:
      return failure(err)

    if err =? self.validatePiece(pieceValidator, blocks).errorOption:
      return failure(err)

    await sleepAsync(1.millis)

  success()
```

We fetch blocks in *batches*, or rather *in pieces*. We trigger fetching blocks with `self.networkStore.getBlock(address)`, which will resolve by either getting the block from the local store or from the network through block delivery described earlier.

Notice that we need to get all the relevant blocks here, not only the blocks that are not yet in the local store. This is necessary, because we need to get all the blocks in a piece so that we can validate the piece and potentially stop streaming if the piece turns out to be invalid.

Before calling `validatePiece`, where validation takes place, we wait for all `Futures` to complete returning the requested `blocks`.

`validatePiece` is defined as follows:

```nim
proc validatePiece(
    self: CodexNodeRef, pieceValidator: TorrentPieceValidator, blocks: seq[bt.Block]
): ?!void {.raises: [].} =
  trace "Fetched complete torrent piece - verifying..."
  let pieceIndex = pieceValidator.validatePiece(blocks)

  if pieceIndex < 0:
    error "Piece verification failed", pieceIndex = pieceIndex
    return failure(fmt"Piece verification failed for {pieceIndex=}")

  trace "Piece successfully verified", pieceIndex

  let confirmedPieceIndex = pieceValidator.confirmCurrentPiece()

  if pieceIndex != confirmedPieceIndex:
    error "Piece confirmation failed",
      pieceIndex = pieceIndex, confirmedPieceIndex = confirmedPieceIndex
    return
      failure(fmt"Piece confirmation failed for {pieceIndex=}, {confirmedPieceIndex=}")
  success()
```

It first calls `validatePiece` on the `pieceValidator`, which computes the SHA1 hash of the concatenated blocks and checks if it matches the (multi) hash from the `info` dictionary.

> [!info]
This constitutes the second validation step: after we checked that the `info` dictionary matches the requested `info` hash in the first step described above, here we are making sure that the received content matches the metadata in the `info` dictionary, and thus it is indeed the content identified by the `info` hash from the request.

`PiecePalidator` maintains internal state so that it known which piece is expected at the given moment - this is why it does not need the piece index argument to validate the blocks. Upon successful validation it returns the index of the validated piece. We then call `pieceValidator.confirmCurrentPiece` to *notify* REST API streaming that is awaiting on `torrentPieceValidator.waitForNextPiece()` before streaming the validated blocks to the requesting client:

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

    let torrentPieceValidator = newTorrentPieceValidator(torrentManifest, codexManifest)

    let stream =
      await node.streamTorrent(torrentManifest, codexManifest, torrentPieceValidator)

    while not stream.atEof:
      trace "Waiting for piece..."
      let pieceIndex = await torrentPieceValidator.waitForNextPiece()

      if -1 == pieceIndex:
        warn "No more torrent pieces expected. TorrentPieceValidator might be out of sync!"
        break

      trace "Got piece", pieceIndex

      let blocksPerPieceIter = torrentPieceValidator.getNewBlocksPerPieceIterator()
      while not blocksPerPieceIter.finished and not stream.atEof:
        var buff = newSeqUninitialized[byte](BitTorrentBlockSize.int)
        # wait for the next the piece to prefetch
        let len = await stream.readOnce(addr buff[0], buff.len)

        buff.setLen(len)
        if buff.len <= 0:
          break

        bytes += buff.len

        await resp.sendChunk(addr buff[0], buff.len)
        discard blocksPerPieceIter.next()
    await resp.finish()
    codex_api_downloads.inc()
  except CancelledError as exc:
    info "Stream cancelled", exc = exc.msg
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
	  await noCancel pieceValidator.cancel()
      await noCancel stream.close()
   except CancelledError:
   trace "Prefetch cancelled"
```

Without this detection mechanism, we would either continue fetching blocks even when streaming API request has been interrupted, or we would continue streaming, even when it is already known that the piece validation phase has failed. This would result in potentially invalid content being returned to the client. After any failure in the `prefetch` job, the pieces will no longer be validated, and thus it does not make any sense to continue the streaming operation.

The `stream.readOnce` called in the streaming loop and implemented in `StoreStream`, which uses the same underlying `networkStore` that is also used in `fetchPieces` proc shown above, will be calling that same `getBlock` operation, which in case the block is not already in local store (because it was already there or as a result of the prefetch operation), will request it from the block exchange engine via `BlockExcEngine.requestBlock` operation. In case there is already a pending request for the given block address, the `PendingBlocksManager` will return the existing block handle, so that  `BlockExcEngine.requestBlock` operation will not cause duplicate request. It will, however potentially return an invalid block to the client, before the containing piece has been validated in the prefetch phase. 

If you want to experiment with uploading and downloading BitTorrent content yourself, check [[Upload and download BitTorrent content with Codex - demo]].