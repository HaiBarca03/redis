# Redis Production Roadmap

> **Mục tiêu tổng thể**: Hiểu Redis đúng cách trong backend production — không chỉ "biết dùng", mà biết **tại sao**, **khi nào**, và **đánh đổi gì**.

**Project Context**: Timekeeping System — NestJS + PostgreSQL + Redis. Scale: 500–2000 employees, peak load 8:00–8:30 AM.

---

## Lộ Trình Học

| Day | Chủ Đề | File |
|---|---|---|
| Day 1 | Redis Foundation & Basic Data Types | [Day1_Redis_Foundation_Production_Guide.md](Day_1/Day1_Redis_Foundation_Production_Guide.md) |
| Day 2 | Redis Core Features | [Day2_Redis_Core_Features_Production_Guide.md](Day_2/Day2_Redis_Core_Features_Production_Guide.md) |
| Day 3 | Redis Patterns | [Day3_Redis_Patterns_Production_Guide.md](Day_3/Day3_Redis_Patterns_Production_Guide.md) |
| Day 4 | Production Redis | [Day4_Redis_Production_Operations_Guide.md](Day_4/Day4_Redis_Production_Operations_Guide.md) |

---

## Day 1 — Redis Foundation & Basic Data Types

**Mục tiêu**: Hiểu Redis là gì, cách tư duy về key design, TTL, và 5 data type cơ bản.

### Các khái niệm chính

- **Redis là gì**: In-memory data structure store — bổ sung PostgreSQL, không thay thế
- **In-memory tradeoffs**: Sub-ms latency, giới hạn RAM, mất data khi restart nếu không config persistence
- **Single-threaded**: INCR/ZADD là atomic; `KEYS *` block hết client — **cấm dùng trong production**
- **Key naming**: `{namespace}:{entity}:{id}` — vd: `cache:employee:100`, `lock:checkin:100:2026-07-08`
- **TTL**: `TTL key` trả về `> 0` (còn X giây), `-1` (không expire), `-2` (key không tồn tại)

### 5 Data Types

| Type | Dùng Cho | Command Chính |
|---|---|---|
| **String** | Cache JSON, counter, OTP, flag | `SET EX`, `GET`, `INCR`, `MGET` |
| **Hash** | Object nhiều field, update từng field | `HSET`, `HGET`, `HGETALL`, `HINCRBY` |
| **List** | Queue FIFO, recent activity | `RPUSH`, `LPOP`, `BRPOP`, `LTRIM` |
| **Set** | Unique collection, membership check | `SADD`, `SISMEMBER`, `SCARD`, `SMEMBERS` |
| **Sorted Set** | Leaderboard, delayed job, sliding rate limit | `ZADD`, `ZRANGE`, `ZRANK`, `ZRANGEBYSCORE` |

### Patterns & Pitfalls

- **DEL vs UNLINK**: Dùng `UNLINK` cho big key (async delete)
- **KEYS vs SCAN**: `KEYS *` cấm, luôn dùng `SCAN` với `COUNT`
- **Cache Stampede**: TTL jitter = `base + random(0, 60)`
- **No TTL on cache**: Memory leak, cache stale mãi mãi

---

## Day 2 — Redis Core Features

**Mục tiêu**: Nắm được các tính năng ngoài data type cơ bản — Bitmap, HLL, Pub/Sub, Streams, Transaction, Lua, Pipeline.

### Tổng Quan

| Feature | Dùng Cho | Không Dùng Cho |
|---|---|---|
| **Bitmap** | Boolean flag theo integer offset (attendance, DAU) | ID lớn/sparse — tốn memory |
| **HyperLogLog** | Đếm unique xấp xỉ (< 0.81% sai số), tiết kiệm memory | Khi cần exact count hoặc lấy danh sách member |
| **Pub/Sub** | Realtime notification nhẹ, mất được | Reliable messaging — dùng Streams |
| **Streams** | Event log có ack, consumer group, retry | Simple use case — overhead lớn hơn List/Pub/Sub |
| **MULTI/EXEC** | Nhóm command atomic không logic | Rollback khi lỗi logic — không có rollback như SQL |
| **WATCH** | Optimistic locking — detect key thay đổi trước EXEC | Throughput cao — WATCH abort rate cao |
| **Lua** | Atomic logic phức tạp (rate limiter, lock release) | Script dài — block event loop |
| **Pipeline** | Giảm network RTT cho bulk ops | Thay thế atomic — pipeline KHÔNG atomic |

### Quy Tắc Chọn

```
Chỉ cần giảm network RTT              → Pipeline
Cần nhóm atomic, không có điều kiện  → MULTI/EXEC
Cần atomic + if/else logic            → Lua
Realtime notification, mất được       → Pub/Sub
Reliable event processing             → Streams
Đếm unique cho million+ items         → HyperLogLog
Boolean flag cho integer ID           → Bitmap
```

### Persistence Intro (học sau Day 4)

- **RDB**: Snapshot định kỳ, restart nhanh, mất nhiều data
- **AOF**: Log mọi command, mất ít data, restart chậm hơn
- **No persistence**: Pure cache, data lấy lại từ DB

---

## Day 3 — Redis Patterns

**Mục tiêu**: Implement các backend pattern thực tế với Redis dùng trong production.

### Cache Patterns

| Pattern | Mô Tả | Khi Dùng |
|---|---|---|
| **Cache-Aside** | Read cache → miss → read DB → set cache | Default cho mỗi cache read |
| **Write Invalidation** | Update DB → DEL cache (không SET) | Tránh poison cache do race condition |
| **TTL Jitter** | TTL = base + random(0, max) | Tránh cache expire đồng loạt |
| **Stale-While-Revalidate** | Trả stale ngay, refresh background | Latency-sensitive endpoint |
| **Null Caching** | Cache kết quả null với TTL ngắn | Chống cache penetration |
| **Mutex per Key** | Lock trước khi rebuild cache | Chống cache stampede |
| **Singleflight** | N concurrent miss → chỉ 1 DB query | Chống stampede, đơn giản hơn mutex |

### 4 Cache Failure Modes

| Mode | Mô Tả | Giải Pháp |
|---|---|---|
| **Stampede** | Popular key expire → hàng trăm request hit DB cùng lúc | TTL jitter, Mutex per key, Singleflight |
| **Penetration** | Query ID không tồn tại → bypass cache, hit DB | Null caching, Input validation, Bloom Filter |
| **Avalanche** | Nhiều key expire cùng lúc → DB quá tải | TTL jitter, Staggered cache warming |
| **Hot Key** | 1 key bị request quá nhiều → Redis bottleneck | L1 local cache, Key sharding |

### Distributed Lock

```
Acquire: SET lock:{resource} {uuid} NX PX {ttl_ms}
Release: Lua script — check uuid → DEL (atomic)
Watchdog: Renew TTL mỗi TTL/3 nếu job dài
```

**Rule**: Redis lock là tầng phòng thủ. **DB UNIQUE constraint là chốt cuối.**

### Idempotency Key

```
Header: X-Idempotency-Key: {uuid}
Redis: SET idempotency:{path}:{key} "processing" NX EX 86400
→ OK  : Request mới, xử lý tiếp
→ nil : Duplicate, trả kết quả cũ hoặc 202
```

### Rate Limiting

| Kiểu | Cách Làm | Ưu Điểm | Nhược Điểm |
|---|---|---|---|
| **Fixed Window** | INCR + EXPIRE (Lua atomic) | Đơn giản, ít memory | Boundary issue (burst 2x limit) |
| **Sliding Window** | ZADD timestamp, ZREMRANGEBYSCORE, ZCARD | Chính xác | Tốn memory hơn (O(1) per request) |

### Queue / Retry / DLQ (Streams)

```
Producer → XADD stream:events * key val
Consumer → XREADGROUP GROUP workers w1 COUNT 10 STREAMS stream:events >
Success  → XACK
Fail     → Pending list
Pending > 30s → XAUTOCLAIM → Retry
Retry > 3x   → XADD DLQ + XACK + Alert
```

### Full Check-in Flow (Bài Tập Lớn)

```
POST /checkin
  [1] Rate Limit:      rate:checkin:{id} → INCR (Lua) → 429 nếu vượt
  [2] Idempotency:     idempotency:checkin:{key} SET NX → skip nếu duplicate
  [3] Acquire Lock:    lock:checkin:{id}:{date} SET NX PX 10000
  [4] Write DB:        INSERT ... ON CONFLICT DO NOTHING
  [5] Invalidate:      UNLINK cache:attendance-summary:{id}:{date}
  [6] Publish:         XADD stream:attendance-events *
  [7] Release Lock:    Lua check uuid → DEL
  [8] Save Idem:       SET idempotency:... {result} EX 86400
  --- Background ---
  [9] Worker:          XREADGROUP → process → XACK
  [10] Retry/DLQ:      XPENDING > 30s → XCLAIM → retry → DLQ
```

---

## Day 4 — Production Redis

**Mục tiêu**: Cấu hình, vận hành, monitor, bảo mật Redis trong production.

### Persistence Decision

| Use Case | Persistence | Lý Do |
|---|---|---|
| Cache (employee, shift, summary) | **None** | Rebuild được từ DB |
| Session, OTP, Idempotency, Stream | **AOF everysec** | Mất = UX xấu hoặc duplicate write |
| Rate limit, lock, counter | **None** | Mất OK, reset tự động |

**Recommendation**: Dùng `aof-use-rdb-preamble yes` (hybrid mode) — restart nhanh + ít mất data.

### Memory & Eviction

```conf
maxmemory 2gb
maxmemory-policy volatile-lru   # Cache key (có TTL) bị evict trước
                                 # Session/lock (TTL dài) được bảo vệ hơn
```

**Alert**: `used_memory > 70% maxmemory`. **Critical**: `> 85%`.

### High Availability

| Option | Khi Dùng | Khả Năng |
|---|---|---|
| **Standalone** | Dev, staging | Không HA |
| **Sentinel** | Production < 10GB, 1 shard | HA, auto failover, không scale-out |
| **Cluster** | > 10GB, write > 100k ops/s | HA + sharding, giới hạn multi-key |
| **Managed** (ElastiCache, Upstash) | Muốn tránh ops burden | HA + monitoring sẵn có |

**Timekeeping**: Sentinel (3 Sentinel + 1 Primary + 2 Replica).

### Hash Tag trong Cluster

```
user:{100}:profile   → hash slot của "100"
user:{100}:session   → hash slot của "100" (cùng slot → MGET được)
```

### Key Metrics Cần Monitor

| Metric | Target | Alert |
|---|---|---|
| `used_memory / maxmemory` | < 70% | > 85% |
| `mem_fragmentation_ratio` | 1.0–1.5 | > 2.0 |
| `evicted_keys` | 0 | > 0 liên tục |
| **Hit ratio** | > 85% | < 80% |
| `replication lag` | 0s | > 5s |
| `rejected_connections` | 0 | > 0 |
| `slowlog entries` | 0 / phút | > 10 / phút |

```bash
# Tính hit ratio
INFO stats | grep keyspace
# hit_ratio = keyspace_hits / (keyspace_hits + keyspace_misses)
```

### Security Baseline

```bash
bind <private-ip>                     # Không expose public
protected-mode yes
ACL SETUSER app on >pass ~cache:* +get +set +del +expire
ACL SETUSER default off               # Disable default user
rename-command FLUSHALL ""            # Disable nguy hiểm
rename-command FLUSHDB ""
```

### Top 7 Operational Pitfalls

1. **Fork lag**: `BGSAVE` trên instance lớn → block vài giây — dùng giờ thấp tải
2. **maxmemory = 0**: Redis dùng hết RAM → OS swap → latency vọt
3. **Không set TTL**: Cache key tồn tại mãi → memory leak → eviction chaos
4. **KEYS * trong production**: Block 200–500ms trên 1M key — dùng `SCAN`
5. **Không monitor SLOWLOG**: Bug `KEYS *` lọt vào production, không phát hiện
6. **Connection pool anti-pattern**: Tạo Redis instance mới mỗi request → 10k connections
7. **Chỉ dựa vào Redis lock**: Không có DB constraint → duplicate data khi Redis fail

---

## Quick Command Reference

```bash
# Key management
TTL key | PERSIST key | UNLINK key
SCAN 0 MATCH cache:* COUNT 100
TYPE key | OBJECT ENCODING key

# Debug / Monitor
MONITOR
SLOWLOG GET 10
MEMORY USAGE key
LATENCY DOCTOR
INFO memory | INFO stats | INFO replication | INFO keyspace

# ACL
ACL LIST | ACL WHOAMI
ACL SETUSER ... | ACL SAVE

# Stream operations
XADD stream * field val
XREADGROUP GROUP grp consumer COUNT 10 STREAMS stream >
XACK stream grp id
XPENDING stream grp - + 10
XAUTOCLAIM stream grp consumer 30000 0-0 COUNT 10

# Lock pattern
SET lock:res uuid NX PX 10000
EVAL "if redis.call('GET',KEYS[1])==ARGV[1] then return redis.call('DEL',KEYS[1]) else return 0 end" 1 lock:res uuid
```

---

## Nguyên Tắc Nhớ Đời

> Redis **bổ sung** PostgreSQL, không **thay thế**.
> Lock Redis là **tranh**. DB constraint là **chốt**.
> Cache hit ratio < 80% → review lại strategy.
> `KEYS *` trong production → incident.
> Không có TTL trên cache key → memory leak.
> Pub/Sub mất message → Streams có ack.
> Redis failure phải **graceful degradation**, không crash app.
