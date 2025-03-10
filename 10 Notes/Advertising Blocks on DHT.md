---
tags:
  - codex/dht-advertising
related:
  - "[[No manifest for BitTorrent on Codex]]"
  - "[[Advertising BitTorrent content on Codex]]"
  - "[[Discovering Blocks on DHT]]"
---
#codex/dht-advertising

| related | [[No manifest for BitTorrent on Codex]], [[No manifest for BitTorrent on Codex]], [[Discovering Blocks on DHT]] |
| ------- | --------------------------------------------------------------------------------------------------------------- |

`Advertiser` (`codex/blockexchange/engine/advertiser.nim`). Here is its constructor:

```nim
proc start*(b: Advertiser) {.async.} =
  ## Start the advertiser
  ##

  trace "Advertiser start"

  proc onBlock(cid: Cid) {.async.} =
    await b.advertiseBlock(cid)

  doAssert(b.localStore.onBlockStored.isNone())
  b.localStore.onBlockStored = onBlock.some

  if b.advertiserRunning:
    warn "Starting advertiser twice"
    return

  b.advertiserRunning = true
  for i in 0 ..< b.concurrentAdvReqs:
    let fut = b.processQueueLoop()
    b.trackedFutures.track(fut)
    asyncSpawn fut

  b.advertiseLocalStoreLoop = advertiseLocalStoreLoop(b)
  b.trackedFutures.track(b.advertiseLocalStoreLoop)
  asyncSpawn b.advertiseLocalStoreLoop
```

Crucial here is `onBlockStored` property of the `localStore`.

`Advertiser` get instance of `RepoStore` as its `localStore` in `CodexServer.new`.

This handler is invoked in `RepoStore.putBlock` and `RepoStore.putBlock` is called from `NetworkStore.putBlock`.

Let's now look quickly at `advertiseBlock`:

```nim
proc advertiseBlock(b: Advertiser, cid: Cid) {.async.} =
  without isM =? cid.isManifest, err:
    warn "Unable to determine if cid is manifest"
    return

  if isM:
    without blk =? await b.localStore.getBlock(cid), err:
      error "Error retrieving manifest block", cid, err = err.msg
      return

    without manifest =? Manifest.decode(blk), err:
      error "Unable to decode as manifest", err = err.msg
      return

    # announce manifest cid and tree cid
    await b.addCidToQueue(cid)
    await b.addCidToQueue(manifest.treeCid)
```

So, the first thing to notice is that **only Manifest Cid are currently advertised**. This may be of crucial importance in the context of [[Advertising BitTorrent content on Codex]] and [[No manifest for BitTorrent on Codex]].

Likewise, also in the `advertiseLocalStoreLoop`, we only track `BlockType.Manifest`:

```nim
proc advertiseLocalStoreLoop(b: Advertiser) {.async: (raises: []).} =
  while b.advertiserRunning:
    try:
      if cids =? await b.localStore.listBlocks(blockType = BlockType.Manifest):
        trace "Advertiser begins iterating blocks..."
        for c in cids:
          if cid =? await c:
            await b.advertiseBlock(cid)
        trace "Advertiser iterating blocks finished."

      await sleepAsync(b.advertiseLocalStoreLoopSleep)
    except CancelledError:
      break # do not propagate as advertiseLocalStoreLoop was asyncSpawned
    except CatchableError as e:
      error "failed to advertise blocks in local store", error = e.msgDetail

  info "Exiting advertise task loop"
```

For each `cid` to be advertised on DHT, the advertiser will use `Discovery.provide` to advertise the `cid` on the DHT using DHT's protocol `addProvider` operation:

```nim
method provide*(d: Discovery, cid: Cid) {.async, base.} =
  ## Provide a block Cid
  ##
  let nodes = await d.protocol.addProvider(cid.toNodeId(), d.providerRecord.get)

  if nodes.len <= 0:
    warn "Couldn't provide to any nodes!"
```

For block discovery on the DHT, please refer to [[Discovering Blocks on DHT]].
