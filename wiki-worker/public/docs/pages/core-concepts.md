# Core Concepts

This page explains the operating modes of go-redis-fallback and which storage tier `Get` / `Set` / `Del` use in each mode.

## Three Storage Tiers

| Tier | Implementation | Lifetime |
|---|---|---|
| Memory cache | `sync.Map` holding `Cache` values | Same as the instance; entries are removed only on expiry or `Del` |
| Redis | `github.com/redis/go-redis/v9` client | Primary store in normal mode |
| Local JSON files | Files under `{DBPath}/{DB}/`, sharded by the key's MD5 | Written only during fallback; deleted after recovery |

Other Go cache libraries also pair an in-process cache with Redis: eko/gocache chains stores such as memory, Redis, and Memcache; go-redis/cache stores MessagePack values in Redis with an optional local in-process cache; mgtv-tech/jetcache-go and cshum/hybridcache build multi-level caches, and cshum/hybridcache adds cache stampede prevention. The third tier here serves a different purpose: local JSON files hold the data written while Redis is unreachable, and the recovery flow pipelines it back to Redis. ledisdb/redis-failover works on the server side, monitoring Redis and performing failover; go-redis-fallback runs inside the application and does not change the Redis deployment.

## Operating Modes

The internal flag `isHealth` (`type.go:57`) selects the mode; each `Get` / `Set` / `Del` reads it once at the start.

| Mode | Entered when | Health check |
|---|---|---|
| Normal | The recovery flow completes (`sync.go:89`) | Not running |
| Fallback | The startup ping fails, or Redis reads/writes fail `MaxRetry` times | Pings every `TimeToCheck` |
| Local-only (no health check) | The startup ping succeeds but recovery aborts because `{DBPath}/{DB}` is missing | Not running; the instance stays here |

The third mode is not a designed state; it results from the recovery flow returning early.

## Operations per Mode

| Operation | Normal mode | Fallback / local-only mode |
|---|---|---|
| `Get` | Memory → Redis (with retries) → on failure, switch and continue on the local path | Memory → local JSON file |
| `Set` | Write Redis (with retries), then memory; on failure, switch and continue on the local path | Write memory and enqueue a batch file write |
| `Del` | Delete from memory, file, and Redis | Delete from memory and file |

## Reading Guide

| To learn | Page |
|---|---|
| How `Get` picks a tier in normal mode and when it degrades | [Normal-Mode Read](/read-fallback) |
| How `Get` restores from files in fallback mode | [Fallback-Mode Read](/read-fallback-local) |
| Why memory hits are written back to Redis | [Read-Repair](/read-repair) |
| When expired data is removed | [Expiry on Read](/read-expiry) |
| When the instance returns to normal mode | [Health Check and Mode Switch](/recovery) |
| Which data is written back to Redis on recovery | [Recovery Resync](/recovery-resync) |
