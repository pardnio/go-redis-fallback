# Configuration

This page lists every field of the `Config` passed to `New`, its default, and how invalid values are corrected.

## Config Fields

### Config

```go
type Config struct {
	Redis  *Redis       `json:"redis"`
	Log    *Log         `json:"log,omitempty"`
	Option *Options     `json:"option,omitempty"`
	Email  *EmailConfig `json:"email,omitempty"`
}
```

| Field | Required | When nil |
|---|---|---|
| `Redis` | **Yes** | `New` panics when the ping succeeds (nil pointer at `sync.go:51`); when the ping fails, the first local file access (a fallback file read or write, `Del`, or the expiry sweep removing a file) panics |
| `Log` | No | All log defaults apply |
| `Option` | No | All option defaults apply |
| `Email` | No | No effect (the current version sends no mail) |

When `Redis` is nil, `initRedis` fills defaults only into its local copy; the instance keeps a nil `Config.Redis`, and the next access to `Redis.DB` panics. Pass `&redisFallback.Redis{}` even when every default is acceptable.

### Redis

| Field | Default | Invalid values |
|---|---|---|
| `Host` | `localhost` | Empty string → `localhost` |
| `Port` | `6379` | `≤ 0` or `> 65535` → `6379` |
| `Password` | `""` | — |
| `DB` | `0` | `< 0` or `> 15` → `0` |

`DB` also names the local file subdirectory `{DBPath}/{DB}`.

### Options

| Field | Default | Effect |
|---|---|---|
| `DBPath` | `./files/redisFallback/db` | Root of the local JSON files; `{DBPath}/{DB}` must exist before startup or normal mode is never entered |
| `MaxRetry` | `3` | Attempts per Redis access in `Get` / `Set`; exhausting them switches to fallback |
| `MaxQueue` | `1000` | Fallback write queue length; when full, `Set` writes the file synchronously |
| `TimeToWrite` | `3s` | Fallback batch write interval; also the maximum delay from `Set` to disk |
| `TimeToCheck` | `1m` | Health-check ping interval; also the maximum delay before a recovered Redis is noticed |

Any numeric field `≤ 0` falls back to its default.

### Log

A type alias of `Log` from `github.com/pardnchiu/go-logger` v0.2.0:

| Field | Default | Description |
|---|---|---|
| `Path` | `./logs/redisFallback` | Log directory containing `debug.log`, `output.log`, `error.log` |
| `Stdout` | `false` | Also write to stdout |
| `MaxSize` | `16 * 1024 * 1024` (16 MiB) | Maximum file size in bytes |
| `MaxBackup` | `5` | Rotated files kept |
| `Type` | `text` | `text` or `json` |

### Email

`EmailConfig` has `Host`, `Port`, `Username`, `Password`, `From`, `To`, `CC`, `Subject`, and `Body` fields, but the send call is commented out in the source (`sync.go:26`), so setting it has no effect.

## Tuning

| Goal | Adjustment |
|---|---|
| Return to normal mode sooner after Redis recovers | Lower `TimeToCheck` |
| Lose less data if the process crashes during fallback | Lower `TimeToWrite` |
| Avoid false fallback switches when Redis is flaky | Raise `MaxRetry` (attempts have no delay between them, so the effect is limited) |

## Full Example

```go
rf, err := redisFallback.New(redisFallback.Config{
	Redis: &redisFallback.Redis{
		Host:     "10.0.0.5",
		Port:     6379,
		Password: os.Getenv("REDIS_PASSWORD"),
		DB:       2,
	},
	Log: &redisFallback.Log{
		Path:   "/var/log/myapp/redis-fallback",
		Stdout: true,
	},
	Option: &redisFallback.Options{
		DBPath:      "/var/lib/myapp/redis-fallback",
		MaxRetry:    3,
		MaxQueue:    5000,
		TimeToWrite: 1 * time.Second,
		TimeToCheck: 15 * time.Second,
	},
})
if err != nil {
	log.Fatal(err)
}
defer rf.Close()
```
