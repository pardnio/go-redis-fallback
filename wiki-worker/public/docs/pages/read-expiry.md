# Expiry on Read

This page explains how TTL is stored, how each tier detects expiry on read, and what the background sweep covers.

## How TTL Is Stored

`Set` converts the TTL to whole seconds in `Cache.TTL` and records the Unix second of the write in `Cache.Timestamp` (`set.go:16-25`).

| `ttl` argument to `Set` | `Cache.TTL` | Result |
|---|---|---|
| `10 * time.Minute` | `600` | Expires after 600 seconds |
| `0` or negative | `0` | Never expires |
| `500 * time.Millisecond` | `0` (truncated) | **Never expires** |
| `1500 * time.Millisecond` | `1` | Expires after 1 second |

A TTL under one second truncates to 0, which means no expiry; Redis is written with 0 as well (no expiry).

## Expiry Check

`isExpired` (`unit.go:26-31`): `TTL ≤ 0` never expires; otherwise the entry is expired when `now > Timestamp + TTL`, at one-second precision.

### Handling per Tier

| Location | On an expired entry | Source |
|---|---|---|
| Normal-mode memory read | Delete the memory entry and the local file, return `Not found` | `get.go:29-34` |
| Fallback-mode memory read | Delete only the memory entry, return `Not found` | `get.go:68-72` |
| Fallback-mode file read | Delete the file, return `Not found` | `get.go:95-100` |
| Recovery resync | Expired entries are not written to Redis | `sync.go:109` |
| Redis itself | Handled by Redis key expiry | — |

## Background Sweep

`New` starts a ticker that fires every 30 seconds (`sync.go:135-154`), iterates the memory cache, and deletes expired entries together with their local files.

| Item | Behavior |
|---|---|
| Scope | Only keys in the memory cache; local files never loaded into memory are not swept |
| Interval | Fixed at 30 seconds, not configurable |
| Stopping | `Close` does not stop this ticker |
| Skip condition | `isRecovering` is checked once when `New` runs; if true the sweep never starts |

### Effect on Reads

`Get` never returns expired data: every tier checks expiry before returning. The background sweep only affects memory and disk usage, not read results.
