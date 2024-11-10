---
tags:
  - bittorrent
---
#bittorrent 

### BitTorrent protocol extensions

If we look at the settings in all three clients, we will easily find something all three BitTorrent client I investigated has some must-haves:

![[protocol-extensions.png]]

All have the following options (all can be enabled or disabled):

- DHT support,
- peer-exchange or PEX,
- Local Peer Discovery
- Encryption

Additionally, Deluge lists here:

- NAT PMP (Port Mapping Protocol),
- UPnP (Universal Plug and Play),

### DHT

In our case DHT needs to be always on unless we want to support centralised trackers. We already discuss some differences in the DHT usage in [[How BitTorrent-Codex integration may look like?]].

### Peer-Exchange or PEX

Peer Exchange (PEX) provides an alternative peer discovery mechanism for swarms once peers have bootstrapped via other mechanisms such as DHT or Tracker announces.

It provides a more up-to-date view of the swarm than most other sources and also reduces the need to query other sources frequently.

We do not seem to have anything like this in Codex... We just use [BitSwap](https://specs.ipfs.tech/bitswap-protocol/).

See [BEP11 - Peer Exchange (PEX)](https://www.bittorrent.org/beps/bep_0011.html).

### Local Service Discovery

Another addition to the BitTorrent protocol that Codex does not seem to have. Local Service Discovery (LSD) or Local Peer Discovery as it is named by sometimes, provides a SSDP-like (http over udp-multicast) mechanism to announce the presence in specific swarms to local neighbours. See [BEP14 - Local Service Discovery](https://www.bittorrent.org/beps/bep_0014.html).

In a way, it is related to NAT PMP.

### NAT PMP (Port Mapping Protocol)

NAT Port Mapping Protocol (NAT-PMP) is a network protocol for establishing network address translation (NAT) settings and port forwarding configurations automatically without user effort.

I do not believe we implement support that in our clients. Do we?

### uTorrent transport protocol (uTP)

What is not shown on the picture above and what is also supported by all BitTorrent clients is [BEP29 - uTorrent transport protocol](https://www.bittorrent.org/beps/bep_0029.html). 

A transport protocol with delay based congestion control. See separate [article](http://www.libtorrent.org/utp.html). The motivation for uTP is for BitTorrent clients to not disrupt internet connections, while still utilizing the unused bandwidth fully.

We do not have this in Codex as far as I know.

### Superseeding

The superseeding ([BEP16](https://www.bittorrent.org/beps/bep_0016.html)) is a feature designed to help a torrent initiator with limited bandwidth to "pump up" a large torrent, reducing the amount of data it needs to upload in order to spawn new seeds in the torrent.

Simply speaking, the node has ability to "focus" on the content it really wants to get out, without loosing bandwidth for multiple smaller seeds. *Super-seed mode is **NOT** recommended for general use.*