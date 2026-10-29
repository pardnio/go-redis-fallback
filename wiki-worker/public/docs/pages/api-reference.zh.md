# API 參考

本頁列出 go-redis-fallback 的建構函式與 `RedisFallback` 方法的簽章、回傳值與錯誤。

## 建構函式

### New

```go
func New(c Config) (*RedisFallback, error)
```

| 項目 | 說明 |
|---|---|
| 行為 | 套用預設值、初始化 logger 與 Redis client、Ping Redis：成功則同步執行[恢復回灌](/zh/recovery-resync)，失敗則進入 fallback 並啟動[健康檢查](/zh/recovery)；最後啟動批次寫檔與 30 秒過期掃除 |
| 錯誤 | 只有 logger 初始化失敗，訊息以 `Failed to initialize` 開頭並以 `%w` 包裝原始錯誤 |
| Redis 無法連線 | 不是錯誤，實例以 fallback 模式回傳 |
| panic | `c.Redis` 為 nil 時：Ping 成功則在 `New` 內，否則在首次存取本地檔時（[說明](/zh/configuration#config)） |

## 方法

### Get

```go
func (rf *RedisFallback) Get(key string) (interface{}, error)
```

| 項目 | 說明 |
|---|---|
| 行為 | 正常模式見 [正常模式讀取](/zh/read-fallback)；fallback 模式見 [Fallback 模式讀取](/zh/read-fallback-local) |
| 回傳值 | 記憶體命中時為寫入時的原始值；從本地檔還原時為 JSON 解碼結果；另一個實例在 Redis 存入 map 或 struct 時為 `nil` 值與 `nil` 錯誤（[說明](/zh/read-fallback#什麼值能從-redis-讀回)） |
| 錯誤 `Not found` | 鍵不存在或已過期 |
| 錯誤 `Failed to parse` | 本地檔內容不是合法 JSON |
| 副作用 | 正常模式記憶體命中時背景寫回 Redis；Redis 讀取失敗時切入 fallback |

### Set

```go
func (rf *RedisFallback) Set(key string, value interface{}, ttl time.Duration) error
```

| 項目 | 說明 |
|---|---|
| 行為 | 見 [寫入路徑](/zh/write-path) |
| `ttl` | 以整數秒儲存；`< 1s` 等同不過期 |
| 錯誤 | `value` 序列化失敗；fallback 佇列已滿且同步寫檔失敗 |
| panic | `value` 為 `nil`（`reflect.TypeOf(nil).String()`） |
| 副作用 | Redis 寫入失敗時切入 fallback |

### Del

```go
func (rf *RedisFallback) Del(key string) error
```

| 項目 | 說明 |
|---|---|
| 行為 | 刪記憶體與本地檔；正常模式另刪 Redis 鍵 |
| 錯誤 | 正常模式下 Redis `DEL` 失敗：`Failed to delete` |
| 不存在的鍵 | 不回傳錯誤 |

### Close

```go
func (rf *RedisFallback) Close()
```

| 停止 | 不停止 |
|---|---|
| 健康檢查 Ticker、批次寫檔 Ticker、Redis client | 批次寫檔 goroutine、30 秒過期掃除 Ticker |

`Close` 不會 flush 佇列中尚未落檔的寫入；呼叫前 `TimeToWrite` 內的 fallback 寫入不會寫到磁碟。

## 型別

`Config`、`Redis`、`Options`、`Cache` 等型別見 [型別參考](/zh/api-reference-types)。
