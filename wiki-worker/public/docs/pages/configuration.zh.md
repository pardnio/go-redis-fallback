# 設定

本頁列出傳入 `New` 的 `Config` 各欄位、預設值，以及無效值如何被修正。

## 設定欄位

### Config

```go
type Config struct {
	Redis  *Redis       `json:"redis"`
	Log    *Log         `json:"log,omitempty"`
	Option *Options     `json:"option,omitempty"`
	Email  *EmailConfig `json:"email,omitempty"`
}
```

| 欄位 | 必要 | 為 nil 時 |
|---|---|---|
| `Redis` | **是** | `New` 在 Ping 成功時 panic（`sync.go:51` nil pointer）；Ping 失敗時首次存取本地檔（fallback 讀檔或落檔、`Del`、過期掃除刪檔）panic |
| `Log` | 否 | 套用全部日誌預設值 |
| `Option` | 否 | 套用全部選項預設值 |
| `Email` | 否 | 無影響（目前版本不寄信） |

`Redis` 為 nil 時，`initRedis` 只在函式內的副本補上預設值，實例保存的 `Config.Redis` 仍是 nil，之後存取 `Redis.DB` 即 panic。即使使用全部預設值，也要傳入 `&redisFallback.Redis{}`。

### Redis

| 欄位 | 預設值 | 無效值處理 |
|---|---|---|
| `Host` | `localhost` | 空字串 → `localhost` |
| `Port` | `6379` | `≤ 0` 或 `> 65535` → `6379` |
| `Password` | `""` | — |
| `DB` | `0` | `< 0` 或 `> 15` → `0` |

`DB` 同時決定本地檔案子目錄 `{DBPath}/{DB}`。

### Options

| 欄位 | 預設值 | 影響 |
|---|---|---|
| `DBPath` | `./files/redisFallback/db` | 本地 JSON 檔根目錄；`{DBPath}/{DB}` 必須在啟動前存在，否則無法進入正常模式 |
| `MaxRetry` | `3` | `Get`／`Set` 存取 Redis 的嘗試次數，用盡即切入 fallback |
| `MaxQueue` | `1000` | Fallback 寫檔佇列長度；佇列滿時 `Set` 改為同步寫檔 |
| `TimeToWrite` | `3s` | Fallback 批次寫檔間隔；也是 `Set` 到落檔的最大延遲 |
| `TimeToCheck` | `1m` | 健康檢查 Ping 間隔；也是 Redis 恢復到被偵測的最大延遲 |

所有數值欄位 `≤ 0` 時套用預設值。

### Log

型別別名自 `github.com/pardnchiu/go-logger` v0.2.0 的 `Log`：

| 欄位 | 預設值 | 說明 |
|---|---|---|
| `Path` | `./logs/redisFallback` | 日誌目錄，內含 `debug.log`、`output.log`、`error.log` |
| `Stdout` | `false` | 同時輸出到標準輸出 |
| `MaxSize` | `16 * 1024 * 1024`（16 MiB） | 單檔大小上限（位元組） |
| `MaxBackup` | `5` | 輪替保留檔數 |
| `Type` | `text` | `text` 或 `json` |

### Email

`EmailConfig` 有 `Host`、`Port`、`Username`、`Password`、`From`、`To`、`CC`、`Subject`、`Body` 欄位，但寄信呼叫在原始碼中被註解（`sync.go:26`），設定後不會有任何效果。

## 調整建議

| 目標 | 調整 |
|---|---|
| Redis 恢復後更快回到正常模式 | 降低 `TimeToCheck` |
| Fallback 期間降低程序崩潰時的資料遺失 | 降低 `TimeToWrite` |
| 降低 Redis 抖動時誤切 fallback | 提高 `MaxRetry`（重試之間沒有等待，效果有限） |

## 完整範例

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
