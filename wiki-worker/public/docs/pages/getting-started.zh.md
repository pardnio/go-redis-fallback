# 快速開始

本頁說明安裝 go-redis-fallback、建立第一個實例，並完成一次讀寫。

## 前置需求

| 項目 | 需求 |
|---|---|
| Go | 1.24.3 或以上（依 `go.mod`） |
| Redis | 任一可連線的 Redis 伺服器；執行期間可以失聯 |
| 檔案系統 | 對 `Options.DBPath`（預設 `./files/redisFallback/db`）與 `Log.Path`（預設 `./logs/redisFallback`）具寫入權限 |

## 安裝

```bash
go get github.com/pardnchiu/go-redis-fallback
```

套件名稱為 `redisFallback`，import 路徑為 `github.com/pardnchiu/go-redis-fallback`。

## 第一次執行

```go
package main

import (
	"fmt"
	"log"
	"os"
	"path/filepath"
	"strconv"
	"time"

	redisFallback "github.com/pardnchiu/go-redis-fallback"
)

func main() {
	dbPath := "./files/redisFallback/db"
	db := 0

	// 首次啟動必須先建立 {DBPath}/{DB}，否則實例不會進入正常模式
	if err := os.MkdirAll(filepath.Join(dbPath, strconv.Itoa(db)), 0755); err != nil {
		log.Fatal(err)
	}

	rf, err := redisFallback.New(redisFallback.Config{
		Redis:  &redisFallback.Redis{Host: "localhost", Port: 6379, DB: db},
		Option: &redisFallback.Options{DBPath: dbPath},
	})
	if err != nil {
		log.Fatal(err)
	}
	defer rf.Close()

	if err := rf.Set("greeting", "hello", 10*time.Minute); err != nil {
		log.Fatal(err)
	}

	value, err := rf.Get("greeting")
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println(value) // hello
}
```

### 確認目前模式

函式庫沒有公開查詢模式的 API，模式切換記錄在日誌中：

| 日誌（`{Log.Path}/output.log`） | 意義 |
|---|---|
| `Starting normal mode` | 啟動時 Ping 成功，開始恢復流程 |
| `[getFromRedis] Switching to fallback mode` | 讀取重試用盡，切入 fallback 模式 |
| `[setToRedis] Switching to fallback mode` | 寫入重試用盡，切入 fallback 模式 |

啟動時 Ping 失敗則記錄在 `error.log`：`Failed to connect, Starting fallback mode`。

## 下一步

- [核心概念](/zh/core-concepts)：三種運作模式與各操作在不同模式下的行為
- [正常模式讀取](/zh/read-fallback)：`Get` 如何降級
- [健康檢查與模式切換](/zh/recovery)：Redis 恢復後何時回到正常模式
