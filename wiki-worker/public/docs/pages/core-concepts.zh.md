# 核心概念

本頁說明 go-redis-fallback 的運作模式，以及 `Get`／`Set`／`Del` 在各模式下走哪一層儲存。

## 三層儲存

| 層 | 實作 | 生命週期 |
|---|---|---|
| 記憶體快取 | `sync.Map`，值為 `Cache` 結構 | 與實例同生命週期；只在過期或 `Del` 時移除 |
| Redis | `github.com/redis/go-redis/v9` client | 正常模式的主要儲存 |
| 本地 JSON 檔 | `{DBPath}/{DB}/` 下以 key 的 MD5 分片的檔案 | 只在 fallback 期間寫入；恢復後刪除 |

其他 Go 快取函式庫同樣把行程內快取與 Redis 搭配使用：eko/gocache 串接 memory、Redis、Memcache 等多種 store；go-redis/cache 以 MessagePack 把值存進 Redis，並可選用本地行程內快取；mgtv-tech/jetcache-go 與 cshum/hybridcache 建構多層快取，cshum/hybridcache 另提供 cache stampede 防護。本專案的第三層用途不同：本地 JSON 檔保存 Redis 無法連線期間寫入的資料，恢復流程再以 Pipeline 寫回 Redis。ledisdb/redis-failover 在伺服器端運作，負責監控 Redis 並執行 failover；go-redis-fallback 則在應用程式內執行，不改變 Redis 的部署架構。

## 運作模式

模式由內部旗標 `isHealth` 決定（`type.go:57`），每次 `Get`／`Set`／`Del` 開頭讀取一次。

| 模式 | 進入條件 | 健康檢查 |
|---|---|---|
| 正常 | 恢復流程完成（`sync.go:89`） | 不執行 |
| Fallback | 啟動 Ping 失敗，或讀寫 Redis 重試 `MaxRetry` 次皆失敗 | 每 `TimeToCheck` Ping 一次 |
| 本地模式（無健康檢查） | 啟動 Ping 成功，但恢復流程因 `{DBPath}/{DB}` 不存在而中止 | 不執行，永遠停留 |

第三種模式不是設計中的狀態，而是恢復流程提前返回的結果。

## 各操作在不同模式下的行為

| 操作 | 正常模式 | Fallback／本地模式 |
|---|---|---|
| `Get` | 記憶體 → Redis（重試）→ 失敗時切換並改走本地路徑 | 記憶體 → 本地 JSON 檔 |
| `Set` | 寫 Redis（重試），成功後寫記憶體；失敗時切換並改走本地路徑 | 寫記憶體並排入批次寫檔 |
| `Del` | 刪記憶體、刪檔、刪 Redis 鍵 | 刪記憶體、刪檔 |

## 本站閱讀路線

| 想知道 | 頁面 |
|---|---|
| 正常模式下 `Get` 如何決定讀哪一層、何時降級 | [正常模式讀取](/zh/read-fallback) |
| Fallback 模式下 `Get` 如何從檔案還原 | [Fallback 模式讀取](/zh/read-fallback-local) |
| 記憶體命中為何會寫回 Redis | [讀取修補](/zh/read-repair) |
| 過期資料何時被移除 | [讀取時過期](/zh/read-expiry) |
| 何時回到正常模式 | [健康檢查與模式切換](/zh/recovery) |
| 恢復時哪些資料寫回 Redis | [恢復回灌](/zh/recovery-resync) |
