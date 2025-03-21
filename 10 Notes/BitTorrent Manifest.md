BitTorrent Manifest is a structure describing BitTorrent content and connecting it to the corresponding Codex content.

It is defined as follows:

```nim
type
  BitTorrentPiece* = MultiHash
  BitTorrentInfo* = ref object
    length* {.serialize.}: uint64
    pieceLength* {.serialize.}: uint32
    pieces* {.serialize.}: seq[BitTorrentPiece]
    name* {.serialize.}: ?string

  BitTorrentManifest* = ref object
    info* {.serialize.}: BitTorrentInfo
    codexManifestCid* {.serialize.}: Cid
```

It contains the BitTorrent `info` dictionary (here only version 1 of the protocol and no support for multiple files and dictionaries).
