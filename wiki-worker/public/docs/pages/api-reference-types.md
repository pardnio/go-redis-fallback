# Type Reference

This page lists the fields and purpose of every exported type in go-redis-fallback.

## Configuration Types

| Type | Purpose | See |
|---|---|---|
| `Config` | The only argument to `New`; holds `Redis`, `Log`, `Option`, `Email` | [Configuration](/configuration) |
| `Redis` | Connection settings: `Host`, `Port`, `Password`, `DB` | [Configuration](/configuration#redis) |
| `Options` | Fallback and recovery parameters: `DBPath`, `MaxRetry`, `MaxQueue`, `TimeToWrite`, `TimeToCheck` | [Configuration](/configuration#options) |
| `Log` | Type alias of `Log` from `github.com/pardnchiu/go-logger` | [Configuration](/configuration#log) |
| `Logger` | Type alias of `Logger` from `github.com/pardnchiu/go-logger`; used internally, never passed in `Config` | — |
| `EmailConfig` | Notification settings; currently inactive | [Configuration](/configuration#email) |

## RedisFallback

The instance type returned by `New`. All fields are unexported; use it only through `Get` / `Set` / `Del` / `Close`. Methods are in the [API Reference](/api-reference).

## Cache

```go
type Cache struct {
	Key       string      `json:"key"`
	Data      interface{} `json:"data"`
	Type      string      `json:"type"`
	Timestamp int64       `json:"timestamp"`
	TTL       int64       `json:"ttl,omitempty"`
}
```

The data structure shared by the memory cache and the local JSON files. In normal mode `Get` also parses Redis values into this structure, but `Set` does not write this structure to Redis; see [Normal-Mode Read](/read-fallback#which-values-read-back-from-redis). Field descriptions are in [Local File Format](/file-storage#file-content).

## Internal Types

These types are exported but only created and used inside the library; callers have no constructor or methods for them:

| Type | Fields | Purpose |
|---|---|---|
| `Writer` | All unexported | The fallback batch writer: queue, pending map, ticker |
| `WriteRequest` | `Key string`, `Data interface{}` | One write sent to the `Writer` queue; `Data` is actually a `Cache` |
| `Path` | All unexported | The file path computed by `getPath` |
