---
tags:
  - bittorrent
urls:
  - https://www.bittorrent.org
  - https://www.bittorrent.com
related-to:
  - "[[Comparison of BitTorrent clients]]"
---
#bittorrent 

| urls       | https://www.bittorrent.org, https://www.bittorrent.com |
| ---------- | ------------------------------------------------------ |
| relates-to | [[Comparison of BitTorrent clients]]                   |

In order to imagine, what do we mean by *BitTorrent - Codex Integration*, we need to gain some basic understanding of BitTorrent. This is no time nor space to create a comprehensive introduction to BitTorrent here (we need to learn it as we go), so, here I just gathered some resources I use to understand the protocol, and by doing it to understand what does it mean for Codex to integrate with BitTorrent clients.

### Specs

BitTorrent spec is build incrementally from so called [BitTorrent Enhancement Proposals (BEPs)](http://bittorrent.org/beps/bep_0000.html). Each BEP adds something to the BitTorrent Protocol. The most important BEPs to study in order to get a good initial grip on the BitTorrent protocol are:

- [[BEP3 - The BitTorrent Protocol Specification]]
- [[BEP52 - The BitTorrent Protocol Specification v2]]
- [[BEP5 - DHT Protocol]]
- [BEP9 - Extension for Peers to Send Metadata Files](https://bittorrent.org/beps/bep_0009.html)
- [BEP10 -Extension Protocol](https://bittorrent.org/beps/bep_0010.html), see also [extension protocol for BitTorrent](https://www.rasterbar.com/products/libtorrent/extension_protocol.html)
- [BEP11 - Peer Exchange (PEX)](https://bittorrent.org/beps/bep_0011.html)
- [BEP23 - Tracker Returns Compact Peer Lists](https://bittorrent.org/beps/bep_0023.html)
- [BEP29 - uTorrent transport protocol](https://bittorrent.org/beps/bep_0029.html)

### libtorrent

[[libtorrent-rasterbar|libtorrent]] is also an excellent source of information about the protocol and it will be the main source of learning the APIs and potential integration points.

> The further you go the more you realise that libtorrent **is** probably the best source of information about the BitTorrent protocol. Perhaps one can even say that most of the protocol developments in the are of BitTorrent happens in libtorrent. Simply speaking, libtorrent is BitTorrent.

### Papers

Selection of some more important BitTorrent papers:

1. [[Incentives Build Robustness in BitTorrent]] - original "BitTorrent" paper by [[Bram Cohen]].
2. [[The Bittorrent P2P File-Sharing System - Measurements And Analysis]] - Paper from the creators of the [[Tribler]] protocol.
### Books

Not so many decent books about the BitTorrent protocol. You can find some *publications* from Springer and occasionally IEEE that focus on some aspects of BitTorrent or its performance.

Below some recommendations:

1. [[The World of Peer-to-Peer (P2P)]]. Community book, free.
2. [[BitTorrent chapter from book P2P and Grids to Services on the Web]].

### BitTorrent Token

Not sure how to categorise this, especially it is just in the area of the BitTorrent ambitions. Yet, clearly, they want to be on the same market as we are: [Whitepaper](https://www.bittorrent.com/btt/btt-docs/BitTorrent_(BTT)_White_Paper_v0.8.7_Feb_2019.pdf).
