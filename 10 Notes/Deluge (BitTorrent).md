---
tags:
  - bittorrent
---
Deluge is a [[Learn BitTorrent|BitTorrent]] client.

Official link: https://deluge-torrent.org.
Git: https://git.deluge-torrent.org/deluge (they are not on GitHub!)

At [https://deluge-torrent.org/about/](https://deluge-torrent.org/about/) we can read that Deluge is able to run on headless machines with the user-interfaces being able to connect remotely from any platform.

From @Giuliano Mega:

I'm also finding that the python bindings are incomplete and expose only a subset of the API btw one way to approach the integration would be by going top down on how Deluge uses libtorrent and then looking at the minimum needed to get it running. The Deluge core is actually a lot simpler than I expected I think this may be less effort than trying to build the entire API from scratch on top of Codex e.g. if we can get enough to run the Deluge daemon on top of Codex, then all the rest (GTK UI, Web UI) sort of works… Other links:

- simple experiment setup: [https://github.com/gmega/bt-experiment](https://github.com/gmega/bt-experiment)
- Deluge fork with instrumentation for metrics: [https://github.com/gmega/deluge](https://github.com/gmega/deluge)
- some notes on libtorrent: [https://hackmd.io/NhVe1A5HT92NALDufacuiA](https://hackmd.io/NhVe1A5HT92NALDufacuiA)
- how to setup a dev env with Deluge + libtorrent: [https://hackmd.io/ESDTgprbSPmViMxc5yKTiQ](https://hackmd.io/ESDTgprbSPmViMxc5yKTiQ)

Related:

- [https://github.com/codex-storage/nim-codex/issues/959](https://github.com/codex-storage/nim-codex/issues/959) - **Codex/BitTorrent integration**
- [https://github.com/codex-storage/nim-codex/issues/951](https://github.com/codex-storage/nim-codex/issues/951) - **Control BitTorrent**
- [Libtorrent and Deluge from sources](https://hackmd.io/ESDTgprbSPmViMxc5yKTiQ)
- [controlling Deluge using its RPC interface](https://github.com/gmega/bt-experiment/blob/c6af36b349f0211df69781233d387de229d68f62/experiment.py#L91)