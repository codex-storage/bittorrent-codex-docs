---
tags:
  - bittorrent
link: http://www.libtorrent.org/
source: https://github.com/arvidn/libtorrent
related-to:
  - "[[Deluge]]"
  - "[[qBittorrent]]"
---
#bittorrent 

| link       | http://www.libtorrent.org/               |
| ---------- | ---------------------------------------- |
| source     | https://github.com/arvidn/libtorrent     |
| related-to | [[Deluge]], [[qBittorrent]] |

Open Source BitTorrent library. The place to go to learn about the internals of the BitTorrent protocol, to find clarifications and answers to question you cannot find elsewhere.

### Selected Docs

- [BitTorrent v2](https://blog.libtorrent.org/2020/09/bittorrent-v2/)- after you know the foundation (e.g. [[BEP3 - The BitTorrent Protocol Specification]], [[BEP52 - The BitTorrent Protocol Specification v2]], [[Learn BitTorrent]]) this already 3 years old document is still probably the best in overview of the changes introduced in the BitTorrent protocol version 2.
- [libtorrent intorduction](https://libtorrent.org/features-ref.html) and some followup links from here: 
	- [DHT Extensions](https://www.libtorrent.org/dht_extensions.html) and [BEP5](https://www.bittorrent.org/beps/bep_0005.html)
	- [Extension Protocol](https://libtorrent.org/extension_protocol.html), [libtorrent overview (aka manual)](https://libtorrent.org/manual-ref.html), [BEP10 - Extension Protocol](https://www.bittorrent.org/beps/bep_0010.html)
	- uTorrent metadata transfer protocol [BEP 9](https://www.bittorrent.org/beps/bep_0009.html) (i.e. magnet links).
	- uTP implementation ([BEP 29](https://www.bittorrent.org/beps/bep_0029.html)). See separate [article](https://libtorrent.org/utp.html).
- [libtorrent tutorial](https://libtorrent.org/tutorial-ref.html) - a good to place to get some intuition on how the library is used.
- [examples](https://libtorrent.org/examples.html) - good as a follow up to the intro.

### Other intersting docs

- [Question about Bittorrent V2 File Hashes #7604](https://github.com/arvidn/libtorrent/discussions/7604) - a good clarification about blocks, padding, and "piece layers".
- From [BitTorrent v2](https://blog.libtorrent.org/2020/09/bittorrent-v2/) mentioned above, we can see how impactful libtorrent is for the BitTorrent protocol version 2: [# Draft: base protocol with merkle trees and new hash algorithms](https://github.com/bittorrent/bittorrent.org/pull/59)

Languages:

![[Pasted image 20241023160207.png]]

### Building

I have followed the instructions from [https://www.libtorrent.org/building.html](https://www.libtorrent.org/building.html).
Relevant sections:
- *building from git*,
- *building with boost build*, and then immediately:
- *Step 4: Installing libtorrent*. 

Documentation not really clear - hard to see the logical order.

### Summary of the building steps

Before anything, install pre-requisites:

```bash
sudo apt install build-essential cmake git ninja-build pkg-config libboost-dev libssl-dev zlib1g-dev libgl1-mesa-dev
```

Then follow with:

```bash
git clone --recurse-submodules https://github.com/arvidn/libtorrent.git
sudo apt install libboost-tools-dev libboost-dev libboost-system-dev
echo "using gcc ;" >>~/user-config.jam
b2 crypto=openssl cxxstd=14 release
sudo b2 install --prefix=/usr/local
```

See also [[libtorrent Python bindings]].