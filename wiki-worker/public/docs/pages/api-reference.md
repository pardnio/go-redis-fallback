# API Reference

This page lists the constructor and `RedisFallback` methods of go-redis-fallback with their signatures, return values, and errors.

## Constructor

### New

```go
func New(c Config) (*RedisFallback, error)
```

| Item | Detail |
|---|---|
| Behavior | Applies defaults, initializes the logger and Redis client, and pings Redis: on success it runs the [Recovery Resync](/recovery-resync) synchronously, on failure it enters fallback and starts the [health check](/recovery); finally it starts the batch writer and the 30-second expiry sweep |
| Errors | Only a logger initialization failure; the message starts with `Failed to initialize` and wraps the cause with `%w` |
| Redis unreachable | Not an error; the instance is returned in fallback mode |
| Panics | When `c.Redis` is nil: inside `New` if the ping succeeds, otherwise on the first local file access ([details](/configuration#config)) |

## Methods

### Get

```go
func (rf *RedisFallback) Get(key string) (interface{}, error)
```

| Item | Detail |
|---|---|
| Behavior | Normal mode: see [Normal-Mode Read](/read-fallback); fallback mode: see [Fallback-Mode Read](/read-fallback-local) |
| Return value | The original value on a memory hit; the JSON-decoded result when restored from a local file; `nil` with a `nil` error when another instance stored a map or struct in Redis ([details](/read-fallback#which-values-read-back-from-redis)) |
| Error `Not found` | The key is missing or expired |
| Error `Failed to parse` | The local file is not valid JSON |
| Side effects | A normal-mode memory hit writes back to Redis in the background; a failed Redis read switches to fallback |

### Set

```go
func (rf *RedisFallback) Set(key string, value interface{}, ttl time.Duration) error
```

| Item | Detail |
|---|---|
| Behavior | See [Write Path](/write-path) |
| `ttl` | Stored as whole seconds; `< 1s` means no expiry |
| Errors | `value` fails to serialize; the fallback queue is full and the synchronous file write fails |
| Panics | When `value` is `nil` (`reflect.TypeOf(nil).String()`) |
| Side effects | A failed Redis write switches to fallback |

### Del

```go
func (rf *RedisFallback) Del(key string) error
```

| Item | Detail |
|---|---|
| Behavior | Deletes from memory and the local file; normal mode also deletes the Redis key |
| Errors | Redis `DEL` fails in normal mode: `Failed to delete` |
| Missing key | No error |

### Close

```go
func (rf *RedisFallback) Close()
```

| Stops | Does not stop |
|---|---|
| Health-check ticker, batch-write ticker, Redis client | Batch-writer goroutine, 30-second expiry sweep ticker |

`Close` does not flush queued writes; fallback writes from the last `TimeToWrite` before the call never reach disk.

## Types

`Config`, `Redis`, `Options`, `Cache`, and the other types are in the [Type Reference](/api-reference-types).
