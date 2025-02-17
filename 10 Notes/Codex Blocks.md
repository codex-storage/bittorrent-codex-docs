---
tags:
  - codex/blocks
related:
  - "[[Codex Block Exchange Protocol]]"
  - "[[Uploading and downloading content in Codex]]"
---

#codex/blocks 

| related | [[Codex Block Exchange Protocol]], [[Uploading and downloading content in Codex]] |
| ------- | --------------------------------------------------------------------------------- |


In the Codex client, blocks are represented by the following data types (`codex/blocktype.nim`):

```nim
type
  Block* = ref object of RootObj
    cid*: Cid
    data*: seq[byte]

  BlockAddress* = object
    case leaf*: bool
    of true:
      treeCid* {.serialize.}: Cid
      index* {.serialize.}: Natural
    else:
      cid* {.serialize.}: Cid
```

And then we have also *block metadata* (`codex/stores/repostore/types.nim`):

```nim
BlockMetadata* {.serialize.} = object
  expiry*: SecondsSince1970
  size*: NBytes
  refCount*: Natural
```
