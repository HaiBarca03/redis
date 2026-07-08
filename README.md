# Redis Production Roadmap

> **Muc tieu tong the**: Hieu Redis dung cach trong backend production — khong chi "biet dung", ma biet **tai sao**, **khi nao**, va **danh doi gi**.

**Project Context**: Timekeeping System — NestJS + PostgreSQL + Redis. Scale: 500–2000 employees, peak load 8:00–8:30 AM.

---

## Lo Trinh Hoc

| Day | Chu De | File |
|---|---|---|
| Day 1 | Redis Foundation & Basic Data Types | [Day1_Redis_Foundation_Production_Guide.md](Day_1/Day1_Redis_Foundation_Production_Guide.md) |
| Day 2 | Redis Core Features | [Day2_Redis_Core_Features_Production_Guide.md](Day_2/Day2_Redis_Core_Features_Production_Guide.md) |
| Day 3 | Redis Patterns | [Day3_Redis_Patterns_Production_Guide.md](Day_3/Day3_Redis_Patterns_Production_Guide.md) |
| Day 4 | Production Redis | [Day4_Redis_Production_Operations_Guide.md](Day_4/Day4_Redis_Production_Operations_Guide.md) |

---

## Day 1 — Redis Foundation & Basic Data Types

**Muc tieu**: Hieu Redis la gi, cach tu duy ve key design, TTL, va 5 data type co ban.

### Cac khai niem chinh

- **Redis la gi**: In-memory data structure store — bo sung PostgreSQL, khong thay the
- **In-memory tradeoffs**: Sub-ms latency, gioi han RAM, mat data khi restart neu khong config persistence
- **Single-threaded**: INCR/ZADD la atomic; `KEYS *` block het client — **cam dung trong production**
- **Key naming**: `{namespace}:{entity}:{id}` — vd: `cache:employee:100`, `lock:checkin:100:2026-07-08`
- **TTL**: `TTL key` tra ve `> 0` (con X giay), `-1` (khong expire), `-2` (key khong ton tai)

### 5 Data Types

| Type | Dung Cho | Command Chinh |
|---|---|---|
| **String** | Cache JSON, counter, OTP, flag | `SET EX`, `GET`, `INCR`, `MGET` |
| **Hash** | Object nhieu field, update tung field | `HSET`, `HGET`, `HGETALL`, `HINCRBY` |
| **List** | Queue FIFO, recent activity | `RPUSH`, `LPOP`, `BRPOP`, `LTRIM` |
| **Set** | Unique collection, membership check | `SADD`, `SISMEMBER`, `SCARD`, `SMEMBERS` |
| **Sorted Set** | Leaderboard, delayed job, sliding rate limit | `ZADD`, `ZRANGE`, `ZRANK`, `ZRANGEBYSCORE` |

### Patterns & Pitfalls

- **DEL vs UNLINK**: Dung `UNLINK` cho big key (async delete)
- **KEYS vs SCAN**: `KEYS *` cam, luon dung `SCAN` voi `COUNT`
- **Cache Stampede**: TTL jitter = `base + random(0, 60)`
- **No TTL on cache**: Memory leak, cache stale mai mai

---

## Day 2 — Redis Core Features

**Muc tieu**: Nam duoc cac tinh nang ngoai data type co ban — Bitmap, HLL, Pub/Sub, Streams, Transaction, Lua, Pipeline.

### Tong Quan

| Feature | Dung Cho | Khong Dung Cho |
|---|---|---|
| **Bitmap** | Boolean flag theo integer offset (attendance, DAU) | ID lon/sparse — ton memory |
| **HyperLogLog** | Dem unique xap xi (< 0.81% sai so), tiet kiem memory | Khi can exact count hoac lay danh sach member |
| **Pub/Sub** | Realtime notification nhe, mat duoc | Reliable messaging — dung Streams |
| **Streams** | Event log co ack, consumer group, retry | Simple use case — overhead lon hon List/Pub/Sub |
| **MULTI/EXEC** | Nhom command atomic khong logic | Rollback khi loi logic — khong co rollback nhu SQL |
| **WATCH** | Optimistic locking — detect key thay doi truoc EXEC | Throughput cao — WATCH abort rate cao |
| **Lua** | Atomic logic phuc tap (rate limiter, lock release) | Script dai — block event loop |
| **Pipeline** | Giam network RTT cho bulk ops | Thay the atomic — pipeline KHONG atomic |

### Quy Tac Chon

```
Chi can giam network RTT              → Pipeline
Can nhom atomic, khong co dieu kien  → MULTI/EXEC
Can atomic + if/else logic            → Lua
Realtime notification, mat duoc       → Pub/Sub
Reliable event processing             → Streams
Dem unique cho million+ items         → HyperLogLog
Boolean flag cho integer ID           → Bitmap
```

### Persistence Intro (hoc sau Day 4)

- **RDB**: Snapshot dinh ky, restart nhanh, mat nhieu data
- **AOF**: Log moi command, mat it data, restart cham hon
- **No persistence**: Pure cache, data lay lai tu DB

---

## Day 3 — Redis Patterns

**Muc tieu**: Implement cac backend pattern thuc te voi Redis dung trong production.

### Cache Patterns

| Pattern | Mo Ta | Khi Dung |
|---|---|---|
| **Cache-Aside** | Read cache → miss → read DB → set cache | Default cho moi cache read |
| **Write Invalidation** | Update DB → DEL cache (khong SET) | Tranh poison cache do race condition |
| **TTL Jitter** | TTL = base + random(0, max) | Tranh cache expire dong loat |
| **Stale-While-Revalidate** | Tra stale ngay, refresh background | Latency-sensitive endpoint |
| **Null Caching** | Cache ket qua null voi TTL ngan | Chong cache penetration |
| **Mutex per Key** | Lock truoc khi rebuild cache | Chong cache stampede |
| **Singleflight** | N concurrent miss → chi 1 DB query | Chong stampede, don gian hon mutex |

### 4 Cache Failure Modes

| Mode | Mo Ta | Giai Phap |
|---|---|---|
| **Stampede** | Popular key expire → hang tram request hit DB cung luc | TTL jitter, Mutex per key, Singleflight |
| **Penetration** | Query ID khong ton tai → bypass cache, hit DB | Null caching, Input validation, Bloom Filter |
| **Avalanche** | Nhieu key expire dong loat → DB qua tai | TTL jitter, Staggered cache warming |
| **Hot Key** | 1 key bi request qua nhieu → Redis bottleneck | L1 local cache, Key sharding |

### Distributed Lock

```
Acquire: SET lock:{resource} {uuid} NX PX {ttl_ms}
Release: Lua script — check uuid → DEL (atomic)
Watchdog: Renew TTL moi TTL/3 neu job dai
```

**Rule**: Redis lock la tang phong thu. **DB UNIQUE constraint la chot cuoi.**

### Idempotency Key

```
Header: X-Idempotency-Key: {uuid}
Redis: SET idempotency:{path}:{key} "processing" NX EX 86400
→ OK  : Request moi, xu ly tiep
→ nil : Duplicate, tra ket qua cu hoac 202
```

### Rate Limiting

| Kieu | Cach Lam | Uu Diem | Nhuoc Diem |
|---|---|---|---|
| **Fixed Window** | INCR + EXPIRE (Lua atomic) | Don gian, it memory | Boundary issue (burst 2x limit) |
| **Sliding Window** | ZADD timestamp, ZREMRANGEBYSCORE, ZCARD | Chinh xac | Ton memory hon (O(1) per request) |

### Queue / Retry / DLQ (Streams)

```
Producer → XADD stream:events * key val
Consumer → XREADGROUP GROUP workers w1 COUNT 10 STREAMS stream:events >
Success  → XACK
Fail     → Pending list
Pending > 30s → XAUTOCLAIM → Retry
Retry > 3x   → XADD DLQ + XACK + Alert
```

### Full Check-in Flow (Bai Tap Lon)

```
POST /checkin
  [1] Rate Limit:      rate:checkin:{id} → INCR (Lua) → 429 neu vuot
  [2] Idempotency:     idempotency:checkin:{key} SET NX → skip neu duplicate
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

**Muc tieu**: Cau hinh, van hanh, monitor, bao mat Redis trong production.

### Persistence Decision

| Use Case | Persistence | Ly Do |
|---|---|---|
| Cache (employee, shift, summary) | **None** | Rebuild duoc tu DB |
| Session, OTP, Idempotency, Stream | **AOF everysec** | Mat = UX xau hoac duplicate write |
| Rate limit, lock, counter | **None** | Mat OK, reset tu dong |

**Recommendation**: Dung `aof-use-rdb-preamble yes` (hybrid mode) — restart nhanh + it mat data.

### Memory & Eviction

```conf
maxmemory 2gb
maxmemory-policy volatile-lru   # Cache key (co TTL) bi evict truoc
                                 # Session/lock (TTL dai) duoc bao ve hon
```

**Alert**: `used_memory > 70% maxmemory`. **Critical**: `> 85%`.

### High Availability

| Option | Khi Dung | Kha Nang |
|---|---|---|
| **Standalone** | Dev, staging | Khong HA |
| **Sentinel** | Production < 10GB, 1 shard | HA, auto failover, khong scale-out |
| **Cluster** | > 10GB, write > 100k ops/s | HA + sharding, gioi han multi-key |
| **Managed** (ElastiCache, Upstash) | Muon tranh ops burden | HA + monitoring san co |

**Timekeeping**: Sentinel (3 Sentinel + 1 Primary + 2 Replica).

### Hash Tag trong Cluster

```
user:{100}:profile   → hash slot cua "100"
user:{100}:session   → hash slot cua "100" (cung slot → MGET duoc)
```

### Key Metrics Can Monitor

| Metric | Target | Alert |
|---|---|---|
| `used_memory / maxmemory` | < 70% | > 85% |
| `mem_fragmentation_ratio` | 1.0–1.5 | > 2.0 |
| `evicted_keys` | 0 | > 0 lien tuc |
| **Hit ratio** | > 85% | < 80% |
| `replication lag` | 0s | > 5s |
| `rejected_connections` | 0 | > 0 |
| `slowlog entries` | 0 / phut | > 10 / phut |

```bash
# Tinh hit ratio
INFO stats | grep keyspace
# hit_ratio = keyspace_hits / (keyspace_hits + keyspace_misses)
```

### Security Baseline

```bash
bind <private-ip>                     # Khong expose public
protected-mode yes
ACL SETUSER app on >pass ~cache:* +get +set +del +expire
ACL SETUSER default off               # Disable default user
rename-command FLUSHALL ""            # Disable nguy hiem
rename-command FLUSHDB ""
```

### Top 7 Operational Pitfalls

1. **Fork lag**: `BGSAVE` tren instance lon → block vai giay — dung gio thap tai
2. **maxmemory = 0**: Redis dung het RAM → OS swap → latency vot
3. **Khong set TTL**: Cache key ton tai mai → memory leak → eviction chaos
4. **KEYS * trong production**: Block 200–500ms tren 1M key — dung `SCAN`
5. **Khong monitor SLOWLOG**: Bug `KEYS *` lay vao production, khong phat hien
6. **Connection pool anti-pattern**: Tao Redis instance moi moi request → 10k connections
7. **Chi dua vao Redis lock**: Khong co DB constraint → duplicate data khi Redis fail

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

## Nguyen Tac Nho Doi

> Redis **bo sung** PostgreSQL, khong **thay the**.
> Lock Redis la **tranh**. DB constraint la **chot**.
> Cache hit ratio < 80% → review lai strategy.
> `KEYS *` trong production → incident.
> Khong co TTL tren cache key → memory leak.
> Pub/Sub mat message → Streams co ack.
> Redis failure phai **graceful degradation**, khong crash app.
