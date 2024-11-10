---
tags:
  - bittorrent
---
#bittorrent 

Recall that we focus on the following three:

- [[qBittorrent]]
- [[Transmission]]
- [[Deluge]]

Below a short summary.

### Feature set

- [[qBittorrent]] appears to be the most comprehensive. Has good support of BitTorrent version 2 features, supports magnet links (`v1`, `v2`, and `hybrid`) has good export capabilities, and provides access to an impressive set of options (both specific to qBittorrent and to [[libtorrent-rasterbar|libtorrent]]). It has the highest user base, looks reasonably good on Ubuntu. 
- [[Transmission]] has best support on macos and looks really good there. Also provides versions based on QT and GTK. User-base similar to that of [[qBittorrent]]. Exposes limited number of settings, besides the most important protocol extensions. On the other hand, less options makes it less overwhelming for a regular user.
- [[Deluge]] - Python-based client which the lowest user base (300k, comparing to around 6M for the other two clients). Feature-wise can be placed somewhere between [[qBittorrent]] and [[Transmission]]. Because it is Phyton, it may feel easier to work with it cross-platform. Yet, feels more buggy than the other two.

Both [[Deluge]] and [[qBittorrent]] depend on [[libtorrent-rasterbar]], while [[Transmission]] is using its own BitTorrent protocol implementation.