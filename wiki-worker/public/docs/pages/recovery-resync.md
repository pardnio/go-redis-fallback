# Recovery Resync

This page explains how data from the fallback period flows from local files through memory back to Redis once Redis recovers, and which entries are not written back.

## Steps

`changeToNormalMode` (`sync.go:50-92`) runs in order:

| Step | Action | On failure |
|---|---|---|
| 1 | Walk every `.json` file under `{DBPath}/{DB}` | Errors such as a missing directory → the whole flow aborts and `isHealth` stays `false` |
| 2 | Read and parse each file as `Cache`, store it in the memory cache under `Cache.Key` | A file that fails to read or parse → error logged, file skipped |
| 3 | `syncMemoryToRedis`: iterate the **entire** memory cache and pipeline entries to Redis | Pipeline execution errors are not checked |
| 4 | `cleanupLocalFile`: delete every `.json` file and empty subdirectory (keeping `{DBPath}/{DB}` itself) | A failed delete → error logged, continue |
| 5 | `isHealth = true` | — |

## Write-Back Rules

`syncMemoryToRedis` (`sync.go:94-133`):

| Item | Rule |
|---|---|
| Scope | Every non-expired entry in the memory cache, including entries that were in memory before fallback |
| Redis TTL | `Timestamp + TTL − now` seconds (remaining time) |
| Remaining TTL ≤ 0 | Not written; entries without a TTL (`ttl = 0`) also land here |
| Batching | One `Exec` per 100 non-expired entries, plus a final `Exec` for the remainder |
| Format | JSON of `Data` (strings have quotes trimmed), same as `Set` |
| Concurrency guard | `isRecovering` uses CAS so only one resync runs at a time; a concurrent call returns immediately |

### Observed Results

Two keys written while Redis was down, then Redis restored:

| Key | Write | Redis after recovery | Local file after recovery |
|---|---|---|---|
| `ttlkey` | `Set("ttlkey", "t", time.Hour)` | Present, TTL 59m59s | Deleted |
| `forever` | `Set("forever", "f", 0)` | Absent | Deleted |

`forever` is still in the memory cache; the first read in normal mode writes it back to Redis through [Read-Repair](/read-repair).

## When Data Survives Only in Memory

Step 4 deletes local files whether or not step 3 succeeded. After any of the following, the data lives only in the memory cache and is lost when the process exits:

| Case | Reason |
|---|---|
| Pipeline `Exec` fails (for example Redis drops again during recovery) | Errors are unchecked and files are deleted anyway |
| A resync is already running | The second `syncMemoryToRedis` call returns immediately, but the file cleanup still runs |
| `ttl = 0` entries | Not in the batch; only read-repair writes them back |

## Memory Cache After Recovery

Recovery does not clear the memory cache. Every loaded entry stays in memory until it expires and is removed by the [background sweep](/read-expiry); in normal mode, `Get` always hits memory for these keys and never asks Redis.
