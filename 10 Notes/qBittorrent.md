---
tags:
  - bittorrent
link: https://www.qbittorrent.org
source: https://github.com/qbittorrent/qBittorrent/
related-to:
  - "[[libtorrent-rasterbar]]"
---
#bittorrent 

| link       | https://www.qbittorrent.org                 |
| ---------- | ------------------------------------------- |
| source     | https://github.com/qbittorrent/qBittorrent/ |
| related-to | [[libtorrent-rasterbar]]                    |

A [[Learn BitTorrent|BitTorrent]] client that is said to be developed by [volunteers](https://www.qbittorrent.org/team) in their spare time.

From Wikipedia:

> **qBittorrent** is a [cross-platform](https://en.wikipedia.org/wiki/Cross-platform "Cross-platform") [free and open-source](https://en.wikipedia.org/wiki/Free_and_open-source "Free and open-source") [BitTorrent client](https://en.wikipedia.org/wiki/BitTorrent_client "BitTorrent client")written in [native](https://en.wikipedia.org/wiki/Native_application "Native application") [C++](https://en.wikipedia.org/wiki/C%2B%2B "C++"). It relies on [Boost](https://en.wikipedia.org/wiki/Boost_(C%2B%2B_libraries) "Boost (C++ libraries)"), [OpenSSL](https://en.wikipedia.org/wiki/OpenSSL "OpenSSL"), [zlib](https://en.wikipedia.org/wiki/Zlib "Zlib"), [Qt](https://en.wikipedia.org/wiki/Qt_(software) "Qt (software)") 6 toolkit and the [[libtorrent-rasterbar]] library (for the torrent back-end), with an optional search engine written in [Python](https://en.wikipedia.org/wiki/Python_(programming_language) "Python (programming language)").[8](https://en.wikipedia.org/wiki/QBittorrent#cite_note-8)[9](https://en.wikipedia.org/wiki/QBittorrent#cite_note-9).

Note about macOS support:

> The macOS version is **barely supported,** because we don't have active macOS developers/contributors. 
> The project is in need of macOS developers. If you are a macOS developer willing to help, just go to our bug tracker for a list of macOS related issues. Or try to fix bugs that you yourself have discovered and annoy you.

I am not sure if that information is completely up-to-date. I did not test building the client on macOS, yet, there seem to be instructions available: [Compilation macOS (x86_64, arm64, cross compilation)](https://github.com/qbittorrent/qBittorrent/wiki/Compilation-macOS-(x86_64,-arm64,-cross-compilation)).

Languages:
	
![[Pasted image 20241023155058.png]]

### How does it look like?

![[qBittorrent-ubuntu.png]]

### Building

Machine: [[Linux Machine]].

Building instructions: [Compilation Debian, Ubuntu, and derivatives](https://github.com/qbittorrent/qBittorrent/wiki/Compilation-Debian,-Ubuntu,-and-derivatives).

When trying to build I was having problems concerning Qt libraries. Following the tutorial, I have installed recommended Qt packages:

```bash
sudo apt install --no-install-recommends qtbase5-dev qttools5-dev libqt5svg5-dev
```

Unfortunately, this did not work. When subsequently running:

```bash
cmake -G "Ninja" -B build -DCMAKE_BUILD_TYPE=RelWithDebInfo -DCMAKE_INSTALL_PREFIX=/usr/local
```

I was getting the following error:

```bash
CMake Error at cmake/Modules/CheckPackages.cmake:49 (find_package):
  By not providing "FindQt6.cmake" in CMAKE_MODULE_PATH this project has
  asked CMake to find a package configuration file provided by "Qt6", but
  CMake did not find one.

  Could not find a package configuration file provided by "Qt6" (requested
  version 6.5.0) with any of the following names:

    Qt6Config.cmake
    qt6-config.cmake

  Add the installation prefix of "Qt6" to CMAKE_PREFIX_PATH or set "Qt6_DIR"
  to a directory containing one of the above files.  If "Qt6" provides a
  separate development package or SDK, be sure it has been installed.
Call Stack (most recent call first):
  CMakeLists.txt:56 (include)
```

To fix this, first needed to [[Installing Qt on Ubuntu 24.04.1 LTS|instal Qt]], then I have modified `./CMakeLists.txt` by adding the following two lines after line `15`:

```cmake
# version requirements - older versions may work, but you are on your own
set(minBoostVersion 1.76)
set(minQt6Version 6.5.0)
set(minOpenSSLVersion 3.0.2)
set(minLibtorrent1Version 1.2.19)
set(minLibtorrentVersion 2.0.10)
set(minZlibVersion 1.2.11)
# added the following two lines:
set(Qt6_DIR "~/Qt/6.8.0/gcc_64/lib/cmake/Qt6/")
set(Qt6GuiTools_DIR "~/Qt/6.8.0/gcc_64/lib/cmake/Qt6GuiTools/")
```

After that I was able to successfully compile the client.
