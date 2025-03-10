---
tags:
  - codex/peer-presence
related:
  - "[[Codex Peer Context Records]]"
---
#codex/peer-presence 

| related | [[Codex Peer Context Records]] |
| ------- | -------------------------------- |
Remote peer's `Presence` records are added to the remote peer context object (`BlockExcPeerCtx`) in one place only: `BlockExcEngine.blockPresenceHandler`.

We remove the records in `BlockExcEngine.blockPresenceHandler` (as part of filtering out irrelevant records), when canceling blocks (in response to receiving new block deliveries), and in the `BlockExcEngine.blockPresenceHandler` itself. 

>[!question]
> isn't the last one (calling `cleanPresence` in `BlockExcEngine.blockPresenceHandler`) a duplicate of `resolveBlocks` which calls `cancelBlocks`, which also calls `cleanPresence` ).`

