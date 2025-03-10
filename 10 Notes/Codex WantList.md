---
tags:
  - codex/want-list
  - codex/block-exchange
related:
  - "[[Codex Block Exchange Protocol]]"
  - "[[Uploading and downloading content in Codex]]"
---
#codex/want-list #codex/block-exchange 

| related | [[Codex Block Exchange Protocol]], [[Uploading and downloading content in Codex]] |
| ------- | --------------------------------------------------------------------------------- |

When engine is being created, it subscribes to the `PeerEventKind.Joined` and `PeerEventKind.Left`:

```nim
network.switch.addPeerEventHandler(peerEventHandler, PeerEventKind.Joined)
network.switch.addPeerEventHandler(peerEventHandler, PeerEventKind.Left)
```

`PeerEventKind.Joined` is triggered when peers connects to us, and `PeerEventKind.Left` when peer disconnects from us.

When peer is joining, we call `setupPeer`

```nim
proc setupPeer*(b: BlockExcEngine, peer: PeerId) {.async.} =
  ## Perform initial setup, such as want
  ## list exchange
  ##

  trace "Setting up peer", peer

  if peer notin b.peers:
    trace "Setting up new peer", peer
    b.peers.add(BlockExcPeerCtx(id: peer))
    trace "Added peer", peers = b.peers.len

  # broadcast our want list, the other peer will do the same
  if b.pendingBlocks.wantListLen > 0:
    trace "Sending our want list to a peer", peer
    let cids = toSeq(b.pendingBlocks.wantList)
    await b.network.request.sendWantList(peer, cids, full = true)

  if address =? b.pricing .? address:
    await b.network.request.sendAccount(peer, Account(address: address))
```

Here is where we send the joining peer our `WantList`.

We get it from `PendingBlocksManager`. `PendingBlocksManager` has a list of *pending* blocks. Every time engine requests a block via `requestBlock(address)`, the block corresponding to the `address` provided becomes pending. It is done via call:

```nim
b.pendingBlocks.getWantHandle(address, b.blockFetchTimeout)
```

`getWantHandle` will put the request address on its `blocks` list, which is a mapping from `BlockAddress` to `BlockReq`:

```nim
p.blocks[address] = BlockReq(
  handle: newFuture[Block]("pendingBlocks.getWantHandle"),
  inFlight: inFlight,
  startTime: getMonoTime().ticks,
)
```

At any given time, the pending blocks form our `WantList` and it will be sent to the joining peer:

```nim
await b.network.request.sendWantList(peer, cids, full = true)
```

where `request.sendWantList` is set to:

```nim
proc sendWantList(
  id: PeerId,
  cids: seq[BlockAddress],
  priority: int32 = 0,
  cancel: bool = false,
  wantType: WantType = WantType.WantHave,
  full: bool = false,
  sendDontHave: bool = false,
): Future[void] {.gcsafe.} =
  self.sendWantList(id, cids, priority, cancel, wantType, full, sendDontHave)
```

in `BlockExcNetwork.new`.

We see that `wantType` argument takes the default value `WantType.WantHave`. The `full` argument is set to `true` in this case, which means this is our full `WantList`.

Thus, intuitively, if a `cid` (`BlockAddress` to be precise) is on the `WantList` with `WantType.WantHave` it means that the corresponding node *wants to have* that cid.

Let's look closer at the `BlockExcEngine.requestBlock` proc:

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

> [!warning]
> `requestBlock` as we see above is undergoing some important changes and for a good reason. First it will be called `downloadInternal` and the `getWantHandle` will no longer be awaiting on returned handle (thus ultimately it will be doing what it days it does). Other important change to notice is that `sendWantHave` will be called only if there are no peers with the requested address; in the version above we see that `WantHave` is sent even if we have a peer with the request address to which we have just sent `WantBlock`.

When a node *requests* a block, we first check if the given pending block has the `inFlight` attribute set, indicating that the block has been recently requested from a remote node known to have it. If it is not the case, we first gather all the peers that have given `cid` and the complementary list of peers that do not have the given `cid`. If no peer in the swarm is having that `cid`, we will trigger discovery. Otherwise, we (pseudo) randomly choose one peer known to have the given `cid` and send it the `WantBlock` request. Subsequently, we then send the `WantHave` request to all the peers known not to have that `cid` (so that they know we are interested in it and let us know that have it once it is the case).

Now, let's look what happens when a peer receives the `WantList`. This is handled by `BlockExcEngine.wantListHandler`:

```nim
proc wantListHandler*(b: BlockExcEngine, peer: PeerId, wantList: WantList) {.async.} =
  let peerCtx = b.peers.get(peer)

  if peerCtx.isNil:
    return

  var
    presence: seq[BlockPresence]
    schedulePeer = false

  for e in wantList.entries:
    let idx = peerCtx.peerWants.findIt(it.address == e.address)

    logScope:
      peer = peerCtx.id
      address = e.address
      wantType = $e.wantType

    if idx < 0: # Adding new entry to peer wants
      let
        have = await e.address in b.localStore
        price = @(b.pricing.get(Pricing(price: 0.u256)).price.toBytesBE)

      case e.wantType
      of WantType.WantHave:
        if have:
          presence.add(
            BlockPresence(
              address: e.address, `type`: BlockPresenceType.Have, price: price
            )
          )
        else:
          if e.sendDontHave:
            presence.add(
              BlockPresence(
                address: e.address, `type`: BlockPresenceType.DontHave, price: price
              )
            )
          peerCtx.peerWants.add(e)

        codex_block_exchange_want_have_lists_received.inc()
      of WantType.WantBlock:
        peerCtx.peerWants.add(e)
        schedulePeer = true
        codex_block_exchange_want_block_lists_received.inc()
    else: # Updating existing entry in peer wants
      # peer doesn't want this block anymore
      if e.cancel:
        trace "Canceling want for block", address = e.address
        peerCtx.peerWants.del(idx)
      else:
        # peer might want to ask for the same cid with
        # different want params
        trace "Updating want for block", address = e.address
        peerCtx.peerWants[idx] = e # update entry

  if presence.len > 0:
    trace "Sending presence to remote", items = presence.mapIt($it).join(",")
    await b.network.request.sendPresence(peer, presence)

  if schedulePeer:
    if not b.scheduleTask(peerCtx):
      warn "Unable to schedule task for peer", peer
```

We go though the `WantList` entries, one-by-one.

1. We check if the `WantList` item is already on the locally kept `WantList` associated with that peer (`peerCtx.peerWants`).
2. If it is not the case, we add new entry to the peer's `WantList`:
	1. We first check if we already have the block corresponding to the `WantList` item in our `localStore`.
	2. If we do, and the `WantList` item is `WantHave`, we add an entry to the `presence` list, otherwise (i.e. when `WantList` item is `WantHave` but we do not have the corresponding block in `localStore`) we add the entry to `peerCtx.peerWants`. If `WantList` item is `WantBlock` we add the corresponding entry to `peerCtx.peerWants` and set a flag to schedule a task where we will eventually send the requested block to the remote peer (we do that even regardless of if we have a block or not in `localStore`).
3. If the `WantList` item is already on the locally kept `WantList` associated with that peer, we just update the entry.