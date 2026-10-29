# 恢復回灌

本頁說明 Redis 恢復後，fallback 期間的資料如何從本地檔案經記憶體寫回 Redis，以及哪些項目不會被寫回。

## 步驟

`changeToNormalMode`（`sync.go:50-92`）依序執行：

| 步驟 | 動作 | 失敗時 |
|---|---|---|
| 1 | 走訪 `{DBPath}/{DB}` 下所有 `.json` 檔 | 目錄不存在等錯誤 → 整個流程中止，`isHealth` 維持 `false` |
| 2 | 逐檔讀取並解析為 `Cache`，以 `Cache.Key` 寫入記憶體快取 | 單檔讀取或解析失敗 → 記錄錯誤、略過該檔 |
| 3 | `syncMemoryToRedis`：遍歷**整個**記憶體快取，以 Pipeline 寫回 Redis | Pipeline 執行錯誤不檢查 |
| 4 | `cleanupLocalFile`：刪除所有 `.json` 檔與空子目錄（保留 `{DBPath}/{DB}` 本身） | 單檔刪除失敗 → 記錄錯誤、繼續 |
| 5 | `isHealth = true` | — |

## 寫回規則

`syncMemoryToRedis`（`sync.go:94-133`）：

| 項目 | 規則 |
|---|---|
| 範圍 | 記憶體快取中所有未過期項目，包含 fallback 前就在記憶體裡的項目 |
| Redis TTL | `Timestamp + TTL − now` 秒（剩餘時間） |
| 剩餘 TTL ≤ 0 | 不寫入；未設 TTL（`ttl = 0`）的項目也落在這裡 |
| 批次 | 每 100 個未過期項目執行一次 `Exec`，最後不足 100 的再執行一次 |
| 寫入格式 | `Data` 的 JSON（字串去掉引號），與 `Set` 相同 |
| 並行保護 | `isRecovering` 以 CAS 確保同時只有一個回灌；已在執行時直接返回 |

### 實測結果

Redis 失聯期間寫入兩個鍵，再讓 Redis 恢復：

| 鍵 | 寫入 | 恢復後 Redis | 恢復後本地檔 |
|---|---|---|---|
| `ttlkey` | `Set("ttlkey", "t", time.Hour)` | 存在，TTL 59m59s | 已刪除 |
| `forever` | `Set("forever", "f", 0)` | 不存在 | 已刪除 |

`forever` 仍在記憶體快取中，正常模式下第一次被讀取時由 [讀取修補](/zh/read-repair) 寫回 Redis。

## 資料只剩記憶體的情況

步驟 4 不論步驟 3 是否成功都會刪除本地檔案。下列情況發生後，資料只存在於記憶體快取，程序結束即遺失：

| 情況 | 原因 |
|---|---|
| Pipeline `Exec` 失敗（如恢復當下 Redis 又斷線） | 錯誤未檢查，檔案照刪 |
| 回灌已在執行中 | 第二次呼叫 `syncMemoryToRedis` 直接返回，但後續清檔照常執行 |
| `ttl = 0` 的項目 | 不在批次中，靠讀取修補才會寫回 |

## 恢復後的記憶體快取

恢復完成後記憶體快取不會被清空。所有載入的項目留在記憶體，直到過期被[背景掃除](/zh/read-expiry)；正常模式的 `Get` 對這些鍵一律記憶體命中，不查 Redis。
