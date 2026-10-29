# 型別參考

本頁列出 go-redis-fallback 所有匯出型別的欄位與用途。

## 設定型別

| 型別 | 用途 | 詳見 |
|---|---|---|
| `Config` | `New` 的唯一參數，包含 `Redis`、`Log`、`Option`、`Email` | [設定](/zh/configuration) |
| `Redis` | 連線設定：`Host`、`Port`、`Password`、`DB` | [設定](/zh/configuration#redis) |
| `Options` | 降級與恢復參數：`DBPath`、`MaxRetry`、`MaxQueue`、`TimeToWrite`、`TimeToCheck` | [設定](/zh/configuration#options) |
| `Log` | `github.com/pardnchiu/go-logger` 的 `Log` 型別別名 | [設定](/zh/configuration#log) |
| `Logger` | `github.com/pardnchiu/go-logger` 的 `Logger` 型別別名；實例內部使用，`Config` 不需要傳入 | — |
| `EmailConfig` | 通知設定；目前未啟用 | [設定](/zh/configuration#email) |

## RedisFallback

`New` 回傳的實例型別，欄位皆未匯出，只能透過 `Get`／`Set`／`Del`／`Close` 操作。方法見 [API 參考](/zh/api-reference)。

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

記憶體快取與本地 JSON 檔共用的資料結構。`Get` 在正常模式下也以此結構解析 Redis 值，但 `Set` 寫入 Redis 的不是此結構，見 [正常模式讀取](/zh/read-fallback#什麼值能從-redis-讀回)。欄位說明見 [本地檔案格式](/zh/file-storage#檔案內容)。

## 內部型別

下列型別雖然匯出，但只由函式庫內部建立與使用，呼叫端沒有可用的建構方式或方法：

| 型別 | 欄位 | 用途 |
|---|---|---|
| `Writer` | 全部未匯出 | Fallback 批次寫檔器：佇列、pending map、Ticker |
| `WriteRequest` | `Key string`、`Data interface{}` | 送入 `Writer` 佇列的單筆寫入；`Data` 實際為 `Cache` |
| `Path` | 全部未匯出 | `getPath` 計算出的檔案路徑 |
