# Getting Started

This page covers installing go-redis-fallback, creating the first instance, and completing one read and write.

## Prerequisites

| Item | Requirement |
|---|---|
| Go | 1.24.3 or higher (per `go.mod`) |
| Redis | Any reachable Redis server; it may go offline at runtime |
| Filesystem | Write access to `Options.DBPath` (default `./files/redisFallback/db`) and `Log.Path` (default `./logs/redisFallback`) |

## Installation

```bash
go get github.com/pardnchiu/go-redis-fallback
```

The package name is `redisFallback`; the import path is `github.com/pardnchiu/go-redis-fallback`.

## First Run

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

	// On a first start {DBPath}/{DB} must exist, otherwise the instance never enters normal mode
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

### Checking the Current Mode

The library exposes no API to query the mode; mode switches are recorded in the log:

| Log line (`{Log.Path}/output.log`) | Meaning |
|---|---|
| `Starting normal mode` | Startup ping succeeded; the recovery flow starts |
| `[getFromRedis] Switching to fallback mode` | Read retries exhausted; switched to fallback mode |
| `[setToRedis] Switching to fallback mode` | Write retries exhausted; switched to fallback mode |

A failed startup ping is recorded in `error.log`: `Failed to connect, Starting fallback mode`.

## Next Steps

- [Core Concepts](/core-concepts): the three operating modes and how each operation behaves in them
- [Normal-Mode Read](/read-fallback): how `Get` degrades
- [Health Check and Mode Switch](/recovery): when the instance returns to normal mode after Redis recovers
