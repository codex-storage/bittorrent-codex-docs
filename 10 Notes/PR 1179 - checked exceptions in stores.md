---
tags:
  - codex/pulls
related-to:
  - "[[Codex PRs]]"
---
https://github.com/codex-storage/nim-codex/pull/1179

Two things to be handled in the follow up PR:

- better signalization in the test `Should propagate cancellation error immediately` in `tests/codex/utils/testsafeasynciter.nim`. See the review comment https://github.com/codex-storage/nim-codex/pull/1179#discussion_r2100831743
- check what to do with `Error: Exception can raise an unlisted exception: Exception` - see discussion in  https://github.com/codex-storage/nim-codex/pull/1179#discussion_r2100873891

For reference, here is the essence of the `Error: Exception can raise an unlisted exception: Exception` thing:

```nim
import pkg/chronos
# the problem does not occur when using std/asyncdispatch
# import std/asyncdispatch

# without "raises: []" it fails with "Error: Exception can raise an unlisted exception: Exception"
#func getIter(): (iterator (): int {.gcsafe.}) =
func getIter(): (iterator (): int {.gcsafe, raises: [].}) =
  return iterator (): int =
    yield 1
    yield 2
    yield 3

proc f1() =
  let iter = getIter()

  proc genNext(): Future[int] {.async.} =
    iter()
```

See also https://github.com/nim-lang/Nim/issues/3772.