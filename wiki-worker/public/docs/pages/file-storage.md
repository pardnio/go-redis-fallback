# Local File Format

This page documents the path rule and content format of the local JSON files written during fallback, for troubleshooting or manual inspection.

## Path Rule

`getPath` (`unit.go:11-24`) shards files into three directory levels by the key's hexadecimal MD5:

```
{DBPath}/{DB}/{md5[0:2]}/{md5[2:4]}/{md5[4:6]}/{md5}.json
```

| Segment | Source |
|---|---|
| `DBPath` | `Options.DBPath`, default `./files/redisFallback/db` |
| `DB` | `Redis.DB`, default `0`; files for different DBs never mix |
| `md5` | The 32-character hexadecimal `md5(key)` |

Example: when the MD5 of key `user:1` is `bdb1dd105679979ca82b28edd1c8ccd2`, the file is `./files/redisFallback/db/0/bd/b1/dd/bdb1dd105679979ca82b28edd1c8ccd2.json`.

## Finding the File for a Key

```bash
key="user:1"
h=$(printf '%s' "$key" | md5)   # on Linux use: md5sum | cut -d' ' -f1
cat "./files/redisFallback/db/0/${h:0:2}/${h:2:2}/${h:4:2}/${h}.json"
```

## File Content

Each file is the JSON of one `Cache` structure:

```json
{
  "key": "user:1",
  "data": {"name": "pardn"},
  "type": "map[string]interface {}",
  "timestamp": 1791139200,
  "ttl": 1800
}
```

| Field | Type | Description |
|---|---|---|
| `key` | string | The original key; the recovery resync uses it as the Redis key |
| `data` | any | The written value |
| `type` | string | `reflect.TypeOf(value).String()` at write time; informational only, unused on read |
| `timestamp` | int64 | Unix seconds at write time |
| `ttl` | int64 | TTL in seconds; omitted when 0, meaning no expiry |

## File Lifecycle

| Event | File |
|---|---|
| `Set` in fallback mode | Written in a batch (or synchronously when the queue is full) |
| `Del` (any mode) | Deleted |
| Expiry found on read | Deleted (normal-mode memory hit, fallback-mode file read) |
| Background sweep finds an expired memory entry | Deleted |
| Recovery resync completes | All deleted, along with empty subdirectories; `{DBPath}/{DB}` is kept |

The `Path` type (`type.go:85-89`) is the return value of `getPath`; all its fields are unexported, so it cannot be constructed or read from outside the package.
