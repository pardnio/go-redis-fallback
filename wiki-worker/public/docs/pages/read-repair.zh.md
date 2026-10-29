# 讀取修補

本頁說明正常模式下記憶體命中時，`Get` 為何會在背景把值寫回 Redis，以及它補上了恢復流程的哪個缺口。

## 行為

正常模式下 `Get` 命中記憶體且未過期時，回傳前會啟動一個 goroutine 執行 `syncToRedis`（`get.go:36`、`sync.go:14-23`）：

| 項目 | 行為 |
|---|---|
| 寫入內容 | `Data` 的 JSON（字串去掉引號），與 `Set` 寫入 Redis 的格式相同 |
| 寫入 TTL | `Cache.TTL` 秒；`TTL` 為 0 時不設過期 |
| 重試 | 無；失敗不記錄、不影響本次回傳 |
| 觸發頻率 | 每次記憶體命中都會寫一次 |

## 為何需要

[恢復回灌](/zh/recovery-resync) 以「剩餘 TTL」寫回 Redis，未設 TTL（`ttl = 0`）的項目算出的剩餘 TTL ≤ 0，會被略過。這些項目恢復後只存在記憶體，第一次被 `Get` 命中時才由讀取修補寫回 Redis。

| 項目 | 恢復批次 | 讀取修補 |
|---|---|---|
| 有 TTL、未過期 | 寫回，TTL = 剩餘秒數 | 再寫一次，TTL = 原始秒數 |
| `ttl = 0` | 略過 | 寫回，不設過期 |

實測：fallback 期間 `Set("forever", "f", 0)`，恢復後 Redis 中沒有 `forever`；正常模式下 `Get("forever")` 一次後，Redis 出現該鍵且無過期時間。

## 寫回特性

| 項目 | 說明 |
|---|---|
| TTL 會被重設 | 讀取修補以原始 `TTL` 秒數寫入，不扣除已經過的時間；頻繁讀取的鍵在 Redis 端的過期時間會一直往後延 |
| 寫入量 | 熱門鍵每次讀取都產生一次 Redis `SET` |
| 從未被讀取的 `ttl = 0` 鍵 | 不會回到 Redis；只要實例存活就留在記憶體 |

## 空項目覆寫 Redis

讀取修補寫回的是記憶體項目當下的 `Data`。另一個實例寫入的 map 或 struct 值從 Redis 讀回時，記憶體項目為 `Data = nil`、`TTL = 0`（見[正常模式讀取](/zh/read-fallback#什麼值能從-redis-讀回)），下一次 `Get` 命中就會把 `null` 以不過期的方式寫入 Redis。

兩個實例連同一個 Redis 的實測：

| 步驟 | 結果 |
|---|---|
| 實例 A：`Set("obj", map[string]any{"name": "pardn"}, time.Hour)` | Redis `obj` = `{"name":"pardn"}`，TTL 1h |
| 實例 B：第一次 `Get("obj")` | 回傳 `nil, nil` |
| 實例 B：第二次 `Get("obj")` | 回傳 `nil, nil`；Redis `obj` 變成 `null`，無 TTL |
