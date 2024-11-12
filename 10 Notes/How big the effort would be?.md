---
tags:
  - bittorrent
related-to:
  - "[[What BitTorrent has that Codex does not]]"
  - "[[How big the effort would be?]]"
---
Depending on the scope. Let's recall the most significant extensions we may need to add to Codex, if we would like Codex client to provide an experience similar to that of BitTorrent:

- Session and Resume Download support,
- Support for Directories and Small Files,
- Bandwidth incentives.
- PEX,
- LSD (Local Service Discovery),
- NAT PMP/UPn{P,
- uTP,
- Superseeding.

See [[What BitTorrent has that Codex does not]] and [[How BitTorrent-Codex integration may look like?]] for more info about these items.

Thus we can only start scoping after we:

1. Revise the Codex peer-exchange protocol and see which BitTorrent extensions we want to adopt in Codex.
2. Settle down on the which Open Source client we want to use.

Once we have this, we can start incrementally realise the Codex/BitTorrent integration. E.g. we can start with integrating Codex as it is and then incrementally add extensions we want to support. This approach would allow us to bootstrap the session management.
