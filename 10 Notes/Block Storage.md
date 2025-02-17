---
tags:
  - codex/block-storage
related:
  - "[[Uploading and downloading content in Codex]]"
  - "[[Codex Blocks]]"
  - "[[Codex Block Exchange Protocol]]"
---
#codex/block-storage

| related | [[Uploading and downloading content in Codex]], [[Codex Blocks]], [[Codex Block Exchange Protocol]] |
| ------- | --------------------------------------------------------------------------------------------------- |

To store blocks and the corresponding metadata, we use `RepoStore`.

`RepoStore` is a proxy to two underlying stores:

- `repoDS` - to store the blocks themselves - by default it is `FSDatastore` as indicated by option `repoKind` in `CodexConf` (`codex/conf.nim`). Other types of storage are also available: `SQLiteDatastore`, `LevelDbDatastore`.
- `metaDS` - to store the blocks' metadata - `LevelDbDatastore` (`vendor/nim-datastore/datastore/leveldb/leveldbds.nim`).

The stores are initialized in `CodexServer.new` (`codex/codex.nim`) and injected into `repoStore` (type `RepoStore` defined in `codex/stores/repostore/types.nim`):

```nim
repoStore = RepoStore.new(
  repoDs = repoData,
  metaDs = LevelDbDatastore.new(config.dataDir / CodexMetaNamespace).expect(
    "Should create metadata store!"
  ),
  quotaMaxBytes = config.storageQuota,
  blockTtl = config.blockTtl,
)
```

The default value for `storageQuota` is given by `config.storageQuota` and `config.blockTtl` (`codex/stores/repostore/types.nim`):

```nim
const
  DefaultBlockTtl* = 24.hours
  DefaultQuotaBytes* = 8.GiBs
```

`repoStore` together with `engine` (`BlockExcEngine`) are parts of `NetworkStore`, which together with `switch`, `engine`, `discovery`, and `prover` is then provided to `codexNode` (`CodexNodeRef`).