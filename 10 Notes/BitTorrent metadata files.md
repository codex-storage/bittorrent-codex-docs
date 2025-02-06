---
tags:
  - bittorrent
related-to:
  - "[[Learn BitTorrent]]"
  - "[[Advertising BitTorrent content on Codex]]"
---
#bittorrent 

| related-to | [[Learn BitTorrent]], [[Advertising BitTorrent content on Codex]] |
| ---------- | ----------------------------------------------------------------- |

Called `torrent` files.

Although we are using trackerless torrents, still the information contained in torrent files is indispensable and will be provided by the peers using number of extension mechanism. We talk more about it in [[Advertising BitTorrent content on Codex]].

## BitTorrent v1

Files are split into fixed-size *pieces* of the same length except for maybe the last one that may be truncated. *Almost always* power of two, most commonly 2^18 = 256 KiB (older versions prior to v 3.2 used 2^20 = 1 MiB as default). For directories, the files are concatenated in the order in which they appear in the file list (see the `files` key in the `info` dictionary below).

Metainfo files, or `.torrent` files are **bencoded** dictionaries with the following keys:

- announce - the url of the tracker
- info - maps to a dictionaries with the following keys:
	- name - suggested name to save a file or dictionary - purely advisory
	- piece length - number of bytes each *piece* is split info

### BitTorrent  v2

Here the `info` dictionary, has the following keys:

- `announce` - the url of the tracker
- `info`
	- `file tree` - a three structure describing each file position in the file tree
		- `length`
		- `pieces root` - Merkle tree toor for the given file
	- `piece length` - as in v1, number of bytes each *piece* is split info
	- `meta version` - set to `2`
	- `name` - as in v1 above
 - `piece layers` - dictionary with entries for each file larger than one piece: key being the corresponding `pieces root` and value being string being a concatenation of piece layer hashes for that file.

We have extensive examples showing how the torrent files are constructed. They can be found under the following url: [https://link.excalidraw.com/readonly/c9z3gLsV1qkTSQJUxEXk](https://link.excalidraw.com/readonly/c9z3gLsV1qkTSQJUxEXk)
