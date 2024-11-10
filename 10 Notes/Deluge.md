---
tags:
  - bittorrent
link: https://deluge-torrent.org
source: https://git.deluge-torrent.org/deluge
related-to:
  - "[[Learn BitTorrent]]"
  - "[[libtorrent-rasterbar]]"
---
#bittorrent 

| link       | https://deluge-torrent.org            |
| ---------- | ------------------------------------- |
| source     | https://git.deluge-torrent.org/deluge |
| related-to | [[Learn BitTorrent]]                  |

Deluge is a [[Learn BitTorrent|BitTorrent]] client.

### Building

I largely follow the instructions from [Setup tutorial for Deluge development](https://deluge.readthedocs.io/en/latest/devguide/tutorials/01-setup.html) with small changes.

First you may need to install:

```bash
sudo apt install intltool closure-compiler
```

> In my case there were already in my system, probably installed with other deps.

My changes come from the fact that I have installed `libtorrent` from sources. See in [[libtorrent-rasterbar]] for the instructions on how to build libtorrent from sources. Then in [[libtorrent Python bindings]] I describe a follow up on how to create the Python bindings for libtorrent. There I create a python environment but I leave the location of the Python virtual environment somehow open.  Now to follow the instructions here, please make sure that the `.venv` folder is at the top-level of your `deluge` directory. In principle, this directory can be put anywhere - and most probably it is better to put it outside of the `deluge` repo: here I follow the practice as given in the above mentioned [Setup tutorial for Deluge development](https://deluge.readthedocs.io/en/latest/devguide/tutorials/01-setup.html).

Install required dependencies:

```bash
sudo apt install python3-geoip python3-dbus  python3-gi python3-gi-cairo gir1.2-gtk-3.0 gir1.2-ayatanaappindicator3-0.1 python3-pygame libnotify4 librsvg2-common xdg-utils
```

> Here, because we have built [[libtorrent-rasterbar|libtorrent]] from source, make sure you omit `python3-libtorrent` from the original instructions.

Then run:

```bash
$ source .venv/bin/activate
(deluge) $ uv pip install -e .
Resolved 22 packages in 1.15s
   Built deluge @ file:///home/codex/code/deluge
   Built rencode==1.0.6
Prepared 21 packages in 5.05s
Installed 21 packages in 9ms
 + attrs==24.2.0
 + automat==24.8.1
 + cffi==1.17.1
 + constantly==23.10.4
 + cryptography==43.0.3
 + deluge==2.1.1.dev127 (from file:///home/codex/code/deluge)
 + hyperlink==21.0.0
 + idna==3.10
 + incremental==24.7.2
 + mako==1.3.6
 + markupsafe==3.0.2
 + pyasn1==0.6.1
 + pyasn1-modules==0.4.1
 + pycparser==2.22
 + pyopenssl==24.2.1
 + pyxdg==0.28
 + rencode==1.0.6
 + service-identity==24.2.0
 + twisted==24.10.0
 + typing-extensions==4.12.2
 + zope-interface==7.1.1
```

Finally in `.venv/pyvenv.cfg` set:

```bash
include-system-site-packages = true
```

Check if it runs:

```bash
deluge-gtk
```

And this is how it looks like:

![[Pasted image 20241107082113.png]]

#### Preferences

- Network:
	![[Pasted image 20241107095618.png]]
	
- Plugins - something specific to Deluge, maybe it is a way to create a *Codex Plugin*.
  This looks like a reach ecosystem: https://deluge-torrent.org/plugins/
	
	![[Pasted image 20241107095922.png]]


Optionally, you may prefer to leave `include-system-site-packages` set to `false`.
Following [this instructions](https://pygobject.gnome.org/getting_started.html#ubuntu-logo-ubuntu-debian-logo-debian):

```bash
$ sudo apt install libgirepository1.0-dev gcc libcairo2-dev pkg-config python3-dev gir1.2-gtk-4.0
$ uv pip install pycairo
$ uv pip install PyGObject
```

We should then be able to uninstall:

```bash
$ sudo apt remove python3-gi python3-gi-cairo gir1.2-gtk-4.0
```

Transmission does support version 2 torrent files but not version 2 magnet links (hybrid magnets are supported).
### Other notes

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