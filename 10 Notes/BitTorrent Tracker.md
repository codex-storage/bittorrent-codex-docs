---
tags:
  - bittorrent
related-to:
  - "[[Learn BitTorrent]]"
---
#bittorrent 

| related-to | [[Learn BitTorrent]] |
| ---------- | -------------------- |
Original BitTorrent tracker coordinates the collaboration in the [[Swarm]]. The trackers is what was making BitTorrent partly centralised - they could be many trackers, but they play the role about the seeders and clients (peers) taking part in the exchange.

With the adoption of DHT in [BEP5 - DHT Protocol](https://www.bittorrent.org/beps/bep_0005.html) and [BEP11 - Peer Exchange (PEX)](https://www.bittorrent.org/beps/bep_0011.html), BitTorrent becomes more decentralised and removes this static point of control: the tracker. See also: [Peer Exchange](https://en.wikipedia.org/wiki/Peer_exchange) protocol in Wikipedia.

A nice note about this can be found in the book [[The World of Peer-to-Peer (P2P)]]:

> Enabling the volatile Peer to operate also as a tracker, but even if this addressed the need for static tracker servers, there is still a centralization of the network around the content. Peers don't have any default ability to contact each other outside of that context.

