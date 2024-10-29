---
tags:
  - bittorrent
link: http://www.libtorrent.org/
source: https://github.com/arvidn/libtorrent
related-to:
  - "[[Deluge (BitTorrent)]]"
  - "[[qBittorrent]]"
---
#bittorrent 

| link       | http://www.libtorrent.org/               |
| ---------- | ---------------------------------------- |
| source     | https://github.com/arvidn/libtorrent     |
| related-to | [[Deluge (BitTorrent)]], [[qBittorrent]] |

Open Source BitTorrent library. 

Languages:

![[Pasted image 20241023160207.png]]

### Building

I have followed the instructions from [https://www.libtorrent.org/building.html](https://www.libtorrent.org/building.html).
I've followed sections *building from git*, *building with boost build*, and then immediately *Step 4: Installing libtorrent*.  Documentation not really clear - hard to see the logical order.

Building steps:

```bash
git clone --recurse-submodules https://github.com/arvidn/libtorrent.git
sudo apt install libboost-tools-dev libboost-dev libboost-system-dev
echo "using gcc ;" >>~/user-config.jam
b2 crypto=openssl cxxstd=14 release
sudo b2 install --prefix=/usr/local
```
