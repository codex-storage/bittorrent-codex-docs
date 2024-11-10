---
tags:
  - bittorrent
---
From [[Learn BitTorrent]] we got a decent overview how BitTorrent protocol looks like.

It turns out there are many similarities between BitTorrent and [Codex](https://docs.codex.storage/learn/whitepaper) at the protocol level.

The following diagram makes the similarity and some differences easier to observe and should be a good introduction to the further discussion:

![[Codex_BitTorrent.svg]]

> If you are browsing this documentation on the web, it might be more convenient to see an online version where you can freely pan and zoom the content: https://link.excalidraw.com/readonly/OOttLqCGKn5smHduI3b9.
> If you are already using obsidian, you can just access the high-res original vector graphic on your file system.

## Discussion

Let's briefly look at the similarities and differences between BitTorrent and Codex.

### Content

BitTorrent clearly focuses on (relatively) small or moderate size content. It also allows to *publish* whole directory tree. Publishing directories remains a relevant feature since it has been subject to deep changes in the version 2 of the protocol.
### Blocks

Both Codex and BitTorrent operate on fixed size blocks on the physical layers. For BitTorrent, the block size is `16KiB` and is fixed. For Codex it is `64KiB` (and is tightly coupled with the storage proofs). Although in the end, what's get exchanged are `16KiB` blocks, `piece`, `request`, `bitfield` and `have` messages operate on so called pieces. The `piece length` is a key in the `info` dictionary and it must be a power of two and at least `16KiB`.

### Manifest

Both protocols use a *manifest*  containing metadata regarding the content.

### DHT

Both Codex and BitTorrent use DHT to locate the content. They do it differently though. 
In Codex, the manifest file and the dataset itself get separate CIDs (Content IDentifiers) that DHT maps to the SPR records of the announcing peers. To discover peers that store (part of) the dataset, a node first asks DHT for the SPR of the peer having the metadata file. In the metadata file, the node finds the CID to the dataset itself (basically a multihash of the corresponding merkle root), and subsequently queries DHT for the SPR of the corresponding node. After that the exchange protocol starts.

In BitTorrent, the `infohash` - a hash of the `info` directory attribute from the manifest (or `.torrent`) file is used to directly query the DHT for the urls of the corresponding nodes. Here, we are particularly interested in the *tracker-less* configuration. In this case, the input to find the content to be downloaded (a file or a directory tree) is the `infohash` which can be delivered to the node out-of-band or, commonly, via a so-called *magnet link*:

```
magnet:?xt=urn:btmh:<tagged-info-hash>
```

Using the `infohash`, the node discovers the relevant peers and then uses protocol extension [BEP-9: Extension for Peers to Send Metadata Files](https://www.bittorrent.org/beps/bep_0009.html). **Supporting protocol extensions may be one of the more important BitTorrent aspects that we may need to address in Codex if we want to remain inter-operable with other clients supporting them.**

### Resuming downloads

In the [libtorrent tutorial](https://www.libtorrent.org/tutorial-ref.html) we can read the following:

> Since bittorrent downloads pieces of files in random order, it's not trivial to resume a partial download. When resuming a download, the bittorrent engine must restore the state of the downloading torrent, specifically which parts of the file(s) are downloaded. There are two approaches to doing this:
> 
> 1. read every piece of the downloaded files from disk and compare it against its expected hash.
> 2. save, to disk, the state of which pieces (and partial pieces) are downloaded, and load it back in again when resuming. 
> 
> If no resume data is provided with a torrent that's added, libtorrent will employ (1) by default.

**Thus, BitTorrent does not have any protocol level for resuming downloads.**

[[libtorrent-rasterbar|libtorrent]] uses so called *sessions* to manage torrent downloads. Together with [torrent_handles](https://www.libtorrent.org/reference-Torrent_Handle.html#torrent_handle) it provides the means to serialise the relevant state data.

### Integrating Codex and BitTorrent

Despite the conceptual similarities between Codex and BitTorrent, trying to perform some surgical cuts in the Codex protocol to make it able to talk to other BitTorrent clients does not seem to be the best way. The most intuitive approach seems to be providing a library with a similar interface as [[libtorrent-rasterbar|libtorrent]] (at least for the clients that depend on it), yet using Codex under the hood. Doing this alone, however, will allow the involved clients to interoperate only if all of them selected "Codex Durability Engine" in the settings instead of using the default BitTorrent protocol. What if a client using the default BitTorrent protocol wants to join a swarm with nodes using Codex protocol?

![[Code-BitTorrent interoperability.svg]]


The most logical (so it seems) approach will be to expect that the user will either:

1. use two separate clients: one for regular BitTorrent downloads, and one for the clients using Codex under the hood. E.g. the users can use regular BitTorrent clients to do best-effort download of the content they are interested in, and then use Codex for more reliable, durable storage options. If we can, with Codex, achieve better performance and stability than BitTorrent, we can gradually see users transitioning to a more reliable protocol.
2. use one client with integrated Codex as an optional setting/plugin. This way, the regular BotTorrent Swarms will be handled with the standard BitTorrent protocol, while, if enabled, at the same time we will be seeding and download the content from the clients having Codex protocol available. Further extending the client with the Marketplace support, we can not only provide durability, but also handle other scenarios like backups, or high-volume storage.

Option (2) seems to be more pragmatic and moreover it makes us more independent from the details of the [[libtorrent-rasterbar|libtorrent]] API (or any other BitTorrent library out there). The following drawing shows three how we could bridge the the two communities: 

![[Code-BitTorrent interoperability-3.svg]]

The nodes in the middle take care for making the content discoverable on both protocols, while the node on the edges are operating using native protocols.

Two DHTs thus. This duplication allows us to keep the original protocols intact. Mostly: we still have to look at things that BitTorrent *has* and we Codex *doesn't* and what other BitTorrent protocol extensions may have serious impact on the relative performance.

Can we use one DHT. No. Unless we want to switch the whole exchange protocol to the one of BitTorrent - and we should have ambition to perform better than BitTorrent.

But imagine, we want to "hot-swap" the BitTorrent protocol with Codex in such a way that the original client (UI/CLI) remains unchanged, supporting all the original BitTorrent options. I fail to find a clear advantage of such approach, especially that it does not seem to improve intra-operability: original BitTorrent clients will still not be able to talk to the *Codexified* BitTorrent clients - their exchange protocols, and their DHTs will be different.

What does make sense though, is take [[libtorrent-rasterbar]] and BitTorrent protocol extensions as a compass to understand what we need to add/change in the Codex API to support the functionalities that BitTorrent users expect. Here, we will have two groups:

1. Things that BitTorrent clients provide and what is not available in Codex - e.g. resuming downloads, session management, or bandwidth utilisation, and - let's call them this way - *integrations* that allow the users to observe and take actions based on the internal protocol events. Some of those things we describe already above, the rest we will cover in the section [[What BitTorrent has that Codex does not]].
2. Low level protocol options penetrating downs to the peer-exchange level. We could take this part as an additional study in order to search for potential improvement points. Some if not all of this work has been done when designing Codex protocol, yet some aspects may have been missed. This part in turn will be covered in [[APIs and protocol optimisations]].