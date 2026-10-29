# 本地檔案格式

本頁說明 fallback 期間本地 JSON 檔的路徑規則與內容格式，供排查或手動檢視使用。

## 路徑規則

`getPath`（`unit.go:11-24`）以 key 的 MD5 十六進位字串分三層目錄：

```
{DBPath}/{DB}/{md5[0:2]}/{md5[2:4]}/{md5[4:6]}/{md5}.json
```

| 片段 | 來源 |
|---|---|
| `DBPath` | `Options.DBPath`，預設 `./files/redisFallback/db` |
| `DB` | `Redis.DB`，預設 `0`；不同 DB 的檔案互不干擾 |
| `md5` | `md5(key)` 的 32 字元十六進位字串 |

範例：key `user:1` 的 MD5 為 `bdb1dd105679979ca82b28edd1c8ccd2` 時，檔案位於 `./files/redisFallback/db/0/bd/b1/dd/bdb1dd105679979ca82b28edd1c8ccd2.json`。

## 從 key 找到檔案

```bash
key="user:1"
h=$(printf '%s' "$key" | md5)   # Linux 改用：md5sum | cut -d' ' -f1
cat "./files/redisFallback/db/0/${h:0:2}/${h:2:2}/${h:4:2}/${h}.json"
```

## 檔案內容

每個檔案是一個 `Cache` 結構的 JSON：

```json
{
  "key": "user:1",
  "data": {"name": "pardn"},
  "type": "map[string]interface {}",
  "timestamp": 1791139200,
  "ttl": 1800
}
```

| 欄位 | 型別 | 說明 |
|---|---|---|
| `key` | string | 原始 key；恢復回灌以此欄位作為 Redis key |
| `data` | any | 寫入的值 |
| `type` | string | 寫入時 `reflect.TypeOf(value).String()`；僅供檢視，讀取時不使用 |
| `timestamp` | int64 | 寫入當下的 Unix 秒數 |
| `ttl` | int64 | TTL 秒數；為 0 時省略，表示不過期 |

## 檔案的生命週期

| 事件 | 檔案 |
|---|---|
| Fallback 模式 `Set` | 批次寫入（或佇列滿時同步寫入） |
| `Del`（任一模式） | 刪除 |
| 讀取時發現過期 | 刪除（正常模式記憶體命中、fallback 模式讀檔） |
| 背景掃除發現記憶體項目過期 | 刪除 |
| 恢復回灌完成 | 全部刪除，連同空子目錄；`{DBPath}/{DB}` 保留 |

`Path` 型別（`type.go:85-89`）是 `getPath` 的回傳值，欄位皆未匯出，外部無法建構或讀取。
