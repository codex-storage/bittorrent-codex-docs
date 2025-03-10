---
tags:
  - codex/peer-presence
related:
  - "[[Codex WantList]]"
---
#codex/peer-presence

| related | [[Codex WantList]], [[When Peer Presence Records are added and removed?]], [[When Peer Want List is updated?]] |
| ------- | -------------------------------------------------------------------------------------------------------------- |

For each remote peer we are interacting with, we create an object of type `BlockExcPeerCtx` (`codex/blockexchange/peers/peercontext.nim`). This object is created in one place only: in `BlockExcEngine.setupPeer`, which is called in response to `PeerEventKind.Joined` peer event (registered in `BlockExcEngine.new`).

`BlockExcPeerCtx` objects keeps two important things about the peer:

1. the blocks the remote peer has - in the form of `Presence` records stored in `blocks: Table[BlockAddress, Presence]`
2. the blocks the remote peer **has explicitly asked for** - as `peerWants: seq[WantListEntry]`

>[!note]
It is easy to get confused, so this note: `peerCtx.peerWants` is a list of blocks for which the remote peer explicitly asked, and not the blocks that remote peer would like to have. The blocks that the remote peer would like to have (but did not request them explicitly yet) are not recorded and handled on-the-fly by sending the presence list in response to `WantHave` request (see `BlockExcEngine.wantListHandler`).

>[!note]
>This is important to emphasize that the *presence list* - the block that the remote peer has, is constrained to the list of blocks that the current peer is interested in (wants). Thus, at any given time, the `blocks` in the `BlockExcPeerCtx` records is conjunction of two sets: the blocks that the remote peer has **AND** the blocks that we want.

>[!info]
>In BitTorrent peers use `have` (equivalent of our `presence` but for single piece only, which may comprise of many blocks) to say they have a block. But then a peer seems to never say which blocks it wants without wanting to download them immediately (so, our `WantBlock`). On the very first message the downloaders send `bitfield` message to indicate which blocks they already have. Downloaders which don't have anything yet may skip the 'bitfield' message. And then they have `interested` `not interested` to indicate if they have interest in receiving blocks (any blocks, those messages do not have payload, so they do not tell which blocks you are or are no longer interested in). Interest state must be kept up to date at all times - whenever a downloader doesn't have something they currently would ask a peer for in `unchoked`, they must express lack of interest, despite being `choked`.

For peer's *have* list, we have a couple of helpers:

- `peerHave` - these are the blocks (given as `seq[BlockAddress]`) the peer has
- `peerHaveCids` - the corresponding cids (`HashSet[Cid]`)
- `contains` - to check if the remote peer `has` given block (as `BlockAddress`)
- `setPresence` and `cleanPresence` to add/remove the blocks from peer's *have* list

For peer's *want* list, there is one helper `peerWantsCids`, giving back the cids of the corresponding blocks (as `HashSet[Cid]`).

See also [[When Peer Presence Records are added and removed?]] and [[When Peer Want List is updated?]]
