# Read-Repair

This page explains why `Get` writes a value back to Redis in the background on a memory hit in normal mode, and which gap in the recovery flow it fills.

## Behavior

In normal mode, when `Get` hits a non-expired memory entry, it starts a goroutine running `syncToRedis` before returning (`get.go:36`, `sync.go:14-23`):

| Item | Behavior |
|---|---|
| Written content | JSON of `Data` (strings have quotes trimmed), the same format `Set` writes to Redis |
| Written TTL | `Cache.TTL` seconds; no expiry when `TTL` is 0 |
| Retries | None; failures are not logged and do not affect the return value |
| Frequency | One write per memory hit |

## Why It Exists

[Recovery Resync](/recovery-resync) writes entries back with their "remaining TTL"; entries without a TTL (`ttl = 0`) compute a remaining TTL ≤ 0 and are skipped. After recovery those entries live only in memory until the first `Get` hit, when read-repair writes them back to Redis.

| Entry | Recovery batch | Read-repair |
|---|---|---|
| Has TTL, not expired | Written, TTL = remaining seconds | Written again, TTL = original seconds |
| `ttl = 0` | Skipped | Written, no expiry |

Observed: `Set("forever", "f", 0)` during fallback leaves no `forever` key in Redis after recovery; one `Get("forever")` in normal mode creates the key in Redis without an expiry.

## Write-Back Characteristics

| Item | Detail |
|---|---|
| TTL is reset | Read-repair writes the original `TTL` seconds without subtracting elapsed time, so frequently read keys keep pushing their Redis expiry forward |
| Write volume | Every read of a hot key issues one Redis `SET` |
| `ttl = 0` keys that are never read | Never return to Redis; they stay in memory for the life of the instance |

## Empty Entries Overwrite Redis

Read-repair writes whatever `Data` the memory entry holds. When another instance's map or struct value was read from Redis, the memory entry has `Data = nil` and `TTL = 0` (see [Normal-Mode Read](/read-fallback#which-values-read-back-from-redis)), so the next `Get` hit writes `null` to Redis with no expiry.

Observed with two instances on the same Redis:

| Step | Result |
|---|---|
| Instance A: `Set("obj", map[string]any{"name": "pardn"}, time.Hour)` | Redis `obj` = `{"name":"pardn"}`, TTL 1h |
| Instance B: first `Get("obj")` | Returns `nil, nil` |
| Instance B: second `Get("obj")` | Returns `nil, nil`; Redis `obj` becomes `null` with no TTL |
