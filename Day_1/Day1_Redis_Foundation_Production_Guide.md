# Day 1: Redis Foundation & Basic Data Types — Production Guide

> **Mindset**: Đây không phải khóa học "Hello World Redis". Đây là cách một senior engineer tư duy khi dùng Redis trong production.

---

## Mục Tiêu

Sau Day 1, bạn phải trả lời được **không cần suy nghĩ**:

- [ ] Redis là gì, khác SQL database ở đâu?
- [ ] In-memory tradeoff: lợi gì, mất gì?
- [ ] Single-threaded có nghĩa là gì trong thực tế?
- [ ] Key naming convention đúng chuẩn production?
- [ ] TTL dùng khi nào, cách đọc TTL -1/-2?
- [ ] String / Hash / List / Set / Sorted Set dùng case nào?
- [ ] Tại sao `KEYS *` bị cấm trong production?
- [ ] DEL vs UNLINK khác nhau thế nào?

**Context Project**: Timekeeping System — NestJS backend, PostgreSQL source of truth, Redis cho cache/session/rate-limit/lock/leaderboard. Scale: ~500-2000 employees, check-in real-time.

---

## Keywords

```
Redis in-memory data structure store
Redis single-threaded event loop
Redis key naming convention (namespace:entity:id)
Redis TTL EXPIRE PERSIST
Redis String Hash List Set Sorted Set
Redis SCAN vs KEYS *
Redis DEL vs UNLINK
Redis atomic command
Redis memory usage estimation
Redis eviction policy
Redis hot key
Redis key cardinality
Redis Big Key
```

---

## 1. Ly Thuyet Cot Loi

### 1.1 Redis La Gi?

Redis = **RE**mote **DI**ctionary **S**erver.

No la **in-memory data structure store** — khong phai "database nhanh". Toan bo data song trong RAM, khong can parse qua disk I/O.

Redis phuc vu cac use case:

| Use Case | Ly do dung Redis |
|---|---|
| Cache | Tranh doc DB lap lai |
| Session store | Fast read, TTL tu xoa |
| OTP / token | TTL tu expire |
| Rate limiter | INCR atomic, khong race condition |
| Distributed lock | SET NX + EX, atomic |
| Leaderboard | Sorted Set rank by score |
| Counter | INCR/DECR atomic |
| Simple queue | List LPUSH/RPOP |
| Online users | Set |
| Delayed job | Sorted Set + score = timestamp |

> **Rule quan trong**: Redis **bo sung** PostgreSQL, khong **thay the**.

---

### 1.2 Redis vs SQL Database

| Tieu chi | PostgreSQL/MySQL | Redis |
|---|---|---|
| Vi tri trong stack | Source of truth | Cache / Coordination layer |
| Query | SQL phuc tap, JOIN | Lookup theo key |
| Transaction | ACID day du | Atomic command, MULTI/EXEC gioi han |
| Data model | Schema, relation | Key-value, data structures |
| Persistence | Durable, WAL | RAM first, persistence optional |
| Scale | Vertical + horizontal (phuc tap) | Horizontal cluster duoc |
| Latency | ms-tens of ms | sub-ms |
| Memory | Disk unlimited | RAM co gioi han |

**Cau ghi nho**:
> PostgreSQL giu **tinh dung lau dai**.
> Redis giu **toc do, trang thai tam, coordination nhe**.

Timekeeping system: PostgreSQL luu attendance record chinh thuc. Redis luu trang thai "employee dang online", "counter check-in hom nay", lock chong duplicate check-in.

---

### 1.3 In-Memory — Tradeoffs That Su

**Uu diem:**
- Latency sub-millisecond (so voi DB: 5-50ms)
- Throughput cuc cao (100k+ ops/sec single instance)
- Data structure phong phu san co

**Nhuoc diem (phai nho ky cho production):**

| Van de | Hau qua | Giai phap |
|---|---|---|
| RAM dat | Khong the luu tat ca data | Chi cache hot data, dung TTL |
| Restart mat data | Neu khong config persistence | RDB + AOF persistence |
| Memory day | Redis tu choi write hoac evict | Eviction policy phu hop |
| Out of Memory | Redis crash | Monitor memory, set `maxmemory` |

**Eviction policy can biet:**

```
noeviction       Từ chối write khi đầy. Nguy hiểm cho cache layer
allkeys-lru      Xóa key ít dùng nhất. Tốt cho cache pure
volatile-lru     Xóa key có TTL ít dùng nhất. Tốt cho mixed workload
volatile-ttl     Xóa key sắp expire sớm nhất
allkeys-random   Xóa random. Tránh dùng
```

**Production timekeeping**: Dùng `allkeys-lru` nếu Redis là pure cache. Dùng `volatile-lru` nếu Redis vừa cache vừa có persistent keys (distributed lock không nên bị evict).

---

### 1.4 Single-Threaded — Y Nghia Thuc Te

Redis xử lý command qua **một main execution thread** duy nhất.

```
Client A: INCR counter:checkin:2026-07-08
Client B: INCR counter:checkin:2026-07-08
```

Kết quả luôn đúng vì Redis xử lý tuần tự. Không bao giờ bị race condition trên single command.

**Hệ quả trong production:**

```
OK  INCR, DECR    Atomic, an toan
OK  SET NX EX     Atomic lock, khong race
OK  ZADD, LPUSH   Atomic
NO  KEYS *        Scan toan bo keyspace, block tat ca client khac
NO  Lua script dai  Block event loop
NO  HGETALL tren hash 10k field  Block
```

> **Rule**: Không bao giờ dùng `KEYS *` trong production. Thay bằng `SCAN`.

```bash
# BAD - block toan bo trong luc scan
KEYS cache:employee:*

# GOOD - scan theo batch, khong block
SCAN 0 MATCH cache:employee:* COUNT 100
```

**Luu y**: Redis 6.0+ có I/O threads xử lý network, nhưng logic xử lý command vẫn single-threaded. Khi nói "single-threaded" = main command processing thread.

---

### 1.5 Key Naming Convention

**Pattern chuan**: `{namespace}:{entity}:{identifier}`

```bash
# Cache
cache:employee:{id}
cache:shift:{id}
cache:attendance-summary:{emp_id}:{date}

# Session / Auth
session:{token}
otp:phone:{phone}
otp:attempt:{phone}

# Rate limiting
rate:login:{user_id}
rate:checkin:{employee_id}

# Distributed Lock
lock:checkin:{employee_id}:{date}
lock:shift-assignment:{shift_id}

# Counter / Stats
stats:checkin:{date}
counter:late-checkin:{date}

# Online tracking
online:employees

# Leaderboard
leaderboard:early-checkin:{date}
```

**Rules quan trong:**

| Rule | Vi du sai | Vi du dung |
|---|---|---|
| Co namespace | `employee:100` | `cache:employee:100` |
| Readable | `e:100:d` | `cache:employee:100` |
| Cache/temp co TTL | `session:abc` khong TTL | `session:abc EX 3600` |
| Khong dung ten chung chung | `data`, `temp`, `user` | `cache:employee:100` |
| Entity + identifier | `employee` | `cache:employee:{id}` |

**Hot key problem**: Nếu hàng nghìn request cùng đọc `cache:employee:1` (giám đốc), đây là hot key. Giải pháp: key sharding `cache:employee:1:shard:{0-9}`.

---

### 1.6 TTL / Expiration

```bash
# Set key voi TTL
SET cache:employee:100 '{"id":100}' EX 300    # 300 seconds
SET otp:phone:0901234567 "123456" PX 120000   # 120000 milliseconds

# Set TTL sau khi key da ton tai
EXPIRE cache:employee:100 300                  # seconds
PEXPIRE cache:employee:100 300000              # milliseconds

# Kiem tra TTL
TTL cache:employee:100    # 297 (con 297 giay)
TTL cache:employee:100    # -1  (khong co expire)
TTL cache:employee:999    # -2  (key khong ton tai)

# Xoa expire
PERSIST cache:employee:100
```

**Bang TTL can nho cho timekeeping:**

| Key | TTL | Ly do |
|---|---|---|
| `cache:employee:{id}` | 5-15 phut | Profile thay doi it |
| `cache:shift:{id}` | 30-60 phut | Shift config it thay doi |
| `session:{token}` | 1-8 gio | Session expire |
| `otp:phone:{phone}` | 2-5 phut | OTP ngan |
| `otp:attempt:{phone}` | 15-60 phut | Rate limit window |
| `rate:login:{id}` | 15-60 phut | Sliding/fixed window |
| `lock:checkin:{id}:{date}` | 10-30 giay | Lock ngan, tu giai phong |
| `stats:checkin:{date}` | 25 gio | Het ngay + buffer |
| `leaderboard:early-checkin:{date}` | 25 gio | Leaderboard hang ngay |
| `online:employees` | Khong TTL | Managed manually |

---

## 2. Data Types — Chi Tiet Production

### 2.1 String

**La gi**: Bytes, toi da 512MB. Binary-safe byte sequence.

**Dung khi**:
- Cache JSON object (serialize)
- Counter (INCR/DECR)
- Flag/boolean
- Token/OTP
- Simple scalar value

```bash
# Cache employee profile
SET cache:employee:100 '{"id":100,"name":"Nguyen Van A","department_id":10}' EX 600
GET cache:employee:100

# Multi-get
MGET cache:employee:100 cache:employee:101 cache:employee:102

# Atomic counter
INCR stats:checkin:2026-07-08        # 1, 2, 3, ...
INCRBY stats:checkin:2026-07-08 5    # Tang 5
DECR quota:otp:100                   # Giam 1

# One-time token: lay va xoa ngay
GETDEL session:one-time-abc
```

**Pattern — Rate limiter (chu y pitfall):**

```bash
INCR rate:login:100
EXPIRE rate:login:100 900
# CAUTION: 2 lenh nay KHONG atomic -> dung Lua neu can chinh xac
```

**Production pitfall**:
- Luu JSON string OK voi cache, nhung neu hay update tung field → dung Hash
- String luu large JSON (>100KB) → ton memory, ton network

---

### 2.2 Hash

**La gi**: Map key-value ben trong mot Redis key. Tuong tu mot row trong SQL.

**Dung khi**:
- Object nhieu field, thuong update tung field
- Khong can TTL rieng cho tung field
- Tiet kiem memory neu field nho (listpack encoding)

```bash
# Luu employee voi nhieu field
HSET employee:100 name "Nguyen Van A" department_id "10" status "active" leave_days_used "2"

# Lay mot field
HGET employee:100 name                     # "Nguyen Van A"

# Lay tat ca field
HGETALL employee:100

# Lay nhieu field cu the
HMGET employee:100 name department_id

# Tang so nguyen trong field (atomic)
HINCRBY employee:100 leave_days_used 1     # 3

# Kiem tra field ton tai
HEXISTS employee:100 status               # 1

# Xoa field
HDEL employee:100 temp_field

# Dem so field
HLEN employee:100
```

**Hash vs String (JSON):**

| Tieu chi | Hash | String (JSON) |
|---|---|---|
| Update tung field thuong xuyen | Tot | Phai doc-parse-update-write lai |
| Read toan bo object | Tot (HGETALL) | Tot (GET) |
| TTL cho ca object | EXPIRE | EX |
| Serialize nested object | Kho | JSON.stringify OK |
| Memory (< 128 fields) | Toi uu listpack | Tuong duong |

**Production Pitfall**: `HGETALL` tren hash voi hang nghin field → Big key, block. Giu hash < 1000 fields.

---

### 2.3 List

**La gi**: Doubly-linked list. Push/pop tu ca 2 dau O(1).

**Dung khi**:
- Simple queue (FIFO: RPUSH + LPOP)
- Stack (LIFO: LPUSH + LPOP)
- Recent activity / history (gioi han bang LTRIM)
- Blocking queue (BRPOP)

```bash
# Queue: producer them job
RPUSH queue:emails "job:email:100"
RPUSH queue:emails "job:email:101"

# Queue: consumer lay theo FIFO
LPOP queue:emails             # "job:email:100"

# Blocking pop: wait toi da 5 giay
BRPOP queue:emails 5

# Recent activity: giu 20 hoat dong gan nhat
LPUSH activity:employee:100 "checkin:2026-07-08T08:00:00"
LTRIM activity:employee:100 0 19

# Xem list
LRANGE activity:employee:100 0 -1   # Tat ca
LRANGE activity:employee:100 0 4    # 5 gan nhat
LLEN activity:employee:100
```

**Production Pitfall**:
- List khong dedup — cung item co the push nhieu lan
- LRANGE 0 -1 tren list lon → Big key, block
- **Redis List khong phai production-grade message queue** — khong co ack, khong retry, khong dead letter. Dung BullMQ (Redis Streams) hoac RabbitMQ/Kafka cho critical work.

---

### 2.4 Set

**La gi**: Unordered collection, khong duplicate, O(1) cho add/remove/membership check.

**Dung khi**:
- Unique collection
- Membership check (ai dang online?)
- Deduplication
- Tag / permission

```bash
# Track employee dang online
SADD online:employees 100 101 102 103

# Kiem tra co online khong
SISMEMBER online:employees 100    # 1 (co)
SISMEMBER online:employees 999    # 0 (khong co)

# Xoa khoi set
SREM online:employees 100

# Dem so luong
SCARD online:employees            # 3

# Lay tat ca (chi dung khi set nho)
SMEMBERS online:employees         # O(N)

# Set operations
SUNION online:employees vip:employees    # Union
SINTER online:employees admin:employees  # Intersection
SDIFF all:employees online:employees     # Difference (ai offline)
```

**Production Pitfall**:
- `SMEMBERS` tren set lon → block. Dung `SSCAN`.
- Set khong luu duoc metadata (join time, score). Neu can → Sorted Set.

---

### 2.5 Sorted Set (ZSet)

**La gi**: Set voi moi member kem mot `score` (float). Luon sorted by score tu dong.

**Dung khi**:
- Leaderboard / ranking
- Priority queue
- Delayed job (score = unix timestamp)
- Time-ordered data
- Rate limit voi sliding window chinh xac

```bash
# Leaderboard: check-in som hom nay
# Score = unix timestamp (thap hon = som hon)
ZADD leaderboard:early-checkin:2026-07-08 1751940000 "employee:100"   # 08:00
ZADD leaderboard:early-checkin:2026-07-08 1751939700 "employee:101"   # 07:55
ZADD leaderboard:early-checkin:2026-07-08 1751940600 "employee:102"   # 08:10

# Top 10 check-in som nhat
ZRANGE leaderboard:early-checkin:2026-07-08 0 9 WITHSCORES

# Top 10 check-in muon nhat (reverse)
ZREVRANGE leaderboard:early-checkin:2026-07-08 0 9 WITHSCORES

# Rank cua employee (0-indexed, 0 = som nhat)
ZRANK leaderboard:early-checkin:2026-07-08 "employee:101"    # 0 (dung dau)

# Score cua mot employee
ZSCORE leaderboard:early-checkin:2026-07-08 "employee:100"

ZCARD leaderboard:early-checkin:2026-07-08
ZREM leaderboard:early-checkin:2026-07-08 "employee:100"

# Delayed Job Queue
ZADD delayed:jobs 1751944200 "send-report:2026-07-08"
ZRANGEBYSCORE delayed:jobs 0 1751944200 LIMIT 0 10
# Sau khi xu ly: ZREM delayed:jobs "send-report:2026-07-08"

# Sliding Window Rate Limit
ZADD rate:api:100 1751940000000 "req:abc"
ZREMRANGEBYSCORE rate:api:100 0 1751939940000   # xoa request ngoai window 60s
ZCARD rate:api:100                              # dem request trong window
```

**Production Pitfall**:
- `ZRANGE` / `ZREVRANGE` deprecated tu Redis 6.2. Dung `ZRANGE ... REV` hoac `ZRANGE ... BYSCORE`.
- Sorted Set khong nen co > 100k members.

---

## 3. Patterns Tong Hop

### 3.1 Distributed Lock (Chong Duplicate Check-in)

```bash
# SET NX EX: atomic "set neu khong ton tai"
SET lock:checkin:100:2026-07-08 "unique-uuid-of-process-a" NX EX 10
# "OK" neu lock thanh cong
# nil neu lock dang bi giu
```

**Release lock dung cach** — PHAI verify value truoc khi DEL:

```lua
-- Lua script atomic (check-and-delete)
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
else
    return 0
end
```

**Ly do can unique value**: Process A set lock, timeout, lock expire, Process B set lock moi. Neu A DEL khong kiem tra value → xoa nham lock cua B.

---

### 3.2 Cache-Aside Pattern voi TTL Jitter

```typescript
// NestJS service example
async getEmployee(id: number): Promise<Employee> {
  const cacheKey = `cache:employee:${id}`;

  const cached = await this.redis.get(cacheKey);
  if (cached) return JSON.parse(cached);

  const employee = await this.employeeRepo.findOne({ where: { id } });
  if (employee) {
    // TTL jitter: tranh cache stampede
    const ttl = 600 + Math.floor(Math.random() * 60);
    await this.redis.set(cacheKey, JSON.stringify(employee), 'EX', ttl);
  }
  return employee;
}
```

---

### 3.3 Rate Limiter — Atomic voi Lua

```lua
-- Lua script atomic
local current = redis.call('GET', KEYS[1])
if current == false then
  redis.call('SET', KEYS[1], 1, 'EX', ARGV[1])
  return 1
else
  return redis.call('INCR', KEYS[1])
end
```

---

### 3.4 DEL vs UNLINK

```bash
DEL cache:employee:100     # Synchronous: block event loop cho den khi xoa xong
UNLINK cache:employee:100  # Async: tra loi ngay, xoa o background thread
```

**Rule production**: Luon dung `UNLINK`. Dac biet quan trong voi big key.

---

## 4. Bai Tap Day 1

### Bai 1: Thiet ke Key Schema

| Use Case | Key | Data Type | TTL | Ly do |
|---|---|---|---|---|
| Employee profile cache | `cache:employee:{id}` | String (JSON) | 600s | It thay doi |
| Shift config cache | `cache:shift:{id}` | String (JSON) | 1800s | Rat it thay doi |
| Attendance summary | `cache:attendance:{emp_id}:{date}` | String (JSON) | 300s | Daily, stale ok |
| Online employees | `online:employees` | Set | No TTL | Managed manually |
| Daily check-in counter | `stats:checkin:{date}` | String (counter) | 25h | Daily reset |
| Early check-in leaderboard | `leaderboard:early-checkin:{date}` | Sorted Set | 25h | Daily |
| OTP code | `otp:phone:{phone}` | String | 120s | Short-lived |
| OTP attempt counter | `otp:attempt:{phone}` | String (counter) | 900s | Rate limit window |
| Login attempts | `rate:login:{user_id}` | String (counter) | 900s | Rate limit |
| Duplicate check-in lock | `lock:checkin:{emp_id}:{date}` | String | 10s | Short lock |
| Recent activity | `activity:employee:{id}` | List | 7d | History |

### Bai 2: Commands Practice (khong nhin tai lieu)

1. Cache employee 100 voi TTL 10 phut
2. Doc employee 100 tu cache
3. Doc dong thoi employee 100, 101, 102
4. Tang counter check-in hom nay (date: 2026-07-08)
5. Set OTP `456789` cho phone `0901234567` expire 2 phut
6. Set employee 200 vao Hash voi field: name, department_id, status
7. Tang leave_days_used cua employee 200 them 1
8. Them employee 100 vao `online:employees`
9. Kiem tra employee 999 co online khong
10. Add leaderboard hom nay: employee 100 check-in luc 08:05 (unix: 1751940300)
11. Lay top 5 check-in som nhat hom nay
12. Scan keys prefix `cache:employee:` ma khong block

---

## 5. Mini Lab

### Lab Setup

```bash
# Chay Redis local voi Docker
docker run -d --name redis-lab -p 6379:6379 redis:7

# Ket noi
docker exec -it redis-lab redis-cli

# Monitor real-time commands (terminal khac)
docker exec -it redis-lab redis-cli MONITOR
```

### Lab 1: Cache Employee Profile

```bash
SET cache:employee:100 '{"id":100,"name":"Nguyen Van A","department_id":10,"status":"active"}' EX 600
GET cache:employee:100
TTL cache:employee:100

# Simulate invalidation khi employee update
DEL cache:employee:100
GET cache:employee:100    # nil (cache miss)
TTL cache:employee:100    # -2 (key khong ton tai)
```

### Lab 2: Daily Check-in Counter

```bash
INCR stats:checkin:2026-07-08
INCR stats:checkin:2026-07-08
INCR stats:checkin:2026-07-08
EXPIRE stats:checkin:2026-07-08 90000

GET stats:checkin:2026-07-08    # "3"
TTL stats:checkin:2026-07-08    # ~90000
```

### Lab 3: Online Employees Tracking

```bash
SADD online:employees 100 101 102 103
SISMEMBER online:employees 100    # 1
SREM online:employees 103
SCARD online:employees            # 3
SMEMBERS online:employees         # {100, 101, 102}
```

### Lab 4: Early Check-in Leaderboard

```bash
ZADD leaderboard:early-checkin:2026-07-08 1751940300 "employee:100"   # 08:05
ZADD leaderboard:early-checkin:2026-07-08 1751939700 "employee:101"   # 07:55
ZADD leaderboard:early-checkin:2026-07-08 1751941200 "employee:102"   # 08:20
ZADD leaderboard:early-checkin:2026-07-08 1751938800 "employee:103"   # 07:40
EXPIRE leaderboard:early-checkin:2026-07-08 90000

# Top 5 som nhat
ZRANGE leaderboard:early-checkin:2026-07-08 0 4 WITHSCORES

# Rank (0-indexed, 0 = som nhat)
ZRANK leaderboard:early-checkin:2026-07-08 "employee:100"     # 2
ZRANK leaderboard:early-checkin:2026-07-08 "employee:103"     # 0 (som nhat)
```

### Lab 5: Distributed Lock Simulation

```bash
# Process A: lock
SET lock:checkin:100:2026-07-08 "process-a-uuid" NX EX 10
# OK

# Process B: co lock cung key
SET lock:checkin:100:2026-07-08 "process-b-uuid" NX EX 10
# nil (that bai)

# Process A: verify roi release
GET lock:checkin:100:2026-07-08    # "process-a-uuid"
DEL lock:checkin:100:2026-07-08    # 1
```

### Lab 6: KEYS vs SCAN

```bash
# Tao 1000 keys test
EVAL "for i=1,1000 do redis.call('SET', 'cache:employee:'..i, 'data', 'EX', 600) end" 0

# BAD - production blocked
KEYS cache:employee:*

# GOOD - cursor-based, an toan
SCAN 0 MATCH cache:employee:* COUNT 100
# Lap voi cursor tra ve cho den khi cursor = 0

INFO keyspace
```

---

## 6. Checklist Day 1

### Kien thuc

- [ ] Giai thich Redis la gi va dung cho gi (khong doc note)
- [ ] Neu 3 tradeoff cua in-memory
- [ ] Giai thich single-threaded va he qua thuc te
- [ ] Thiet ke key name cho 5 use case bat ky
- [ ] Giai thich TTL -1 va TTL -2
- [ ] Neu data type nao dung cho leaderboard va tai sao
- [ ] Giai thich tai sao KEYS * bi cam

### Thuc hanh

- [ ] Chay Redis local bang Docker
- [ ] CRUD voi String, Hash, List, Set, Sorted Set
- [ ] Set va verify TTL
- [ ] Chay SCAN thay KEYS *
- [ ] Simulate distributed lock voi SET NX EX
- [ ] Dung MONITOR de xem real-time commands

---

## 7. Production Pitfalls

### P1: KEYS * trong Production

```bash
# Tuyet doi khong dung
KEYS *
KEYS cache:*

# Luon dung SCAN
SCAN 0 MATCH cache:employee:* COUNT 100
```

**Hau qua thuc te**: KEYS * tren Redis 1 trieu key co the block 200-500ms. Tat ca request khac bi treo. Production down.

---

### P2: Big Key

Big key = value rat lon, hoac collection co qua nhieu member.

```bash
# Detect big key
redis-cli --bigkeys

# Memory cua key cu the
MEMORY USAGE cache:employee:100
```

**Rule**:
- String: < 1MB OK, > 10MB can review
- Hash/List/Set/ZSet: < 10,000 members OK

---

### P3: Cache Key Khong Co TTL

```bash
# Memory leak
SET cache:employee:100 '{"id":100}'

# Dung
SET cache:employee:100 '{"id":100}' EX 600
```

**Hau qua**: Memory tang mai, eviction kick in, cache stale vinh vien.

---

### P4: Cache Stampede (Thundering Herd)

**Khi nao**: Cache expire dong loat cua popular key → tat ca request hit DB → DB qua tai.

**Giai phap**:
- TTL jitter: `EX (base + random(0, 60))` — tranh expire dong loat
- Probabilistic early refresh: refresh truoc khi expire
- Lock pattern: chi 1 request rebuild cache, cac request khac dung stale hoac doi

---

### P5: Race Condition trong Rate Limiter

```bash
# INCR + EXPIRE khong atomic
INCR rate:login:100
EXPIRE rate:login:100 900
# Neu INCR thanh cong nhung process crash → key khong co TTL → memory leak
```

**Giai phap**: Lua script atomic (xem Section 3.3).

---

### P6: DEL vs UNLINK cho Big Key

```bash
# Block synchronously
DEL big-hash-100k-fields

# Async background delete
UNLINK big-hash-100k-fields
```

---

### P7: Lock Renewal cho Long-running Job

**Van de**: Job chay 30s nhung lock chi 10s → lock expire giua chung → race condition.

```typescript
// Watchdog: renew lock moi 5s
const renewInterval = setInterval(async () => {
  await redis.expire(`lock:checkin:${empId}:${date}`, 10);
}, 5000);

try {
  await processCheckin();
} finally {
  clearInterval(renewInterval);
  await redis.del(`lock:checkin:${empId}:${date}`);
}
```

---

## 8. Senior Review Questions

**Q1**: Ban cache employee profile bang String JSON. Team leader hoi: "Sao khong dung Hash?"

> **Answer**: String tot khi read toan bo object mot lan, it update individual field. Hash tot khi thuong xuyen HINCRBY hoac HSET tung field. Profile thuong chi doc → String simpler, it overhead. Neu co field nhu `leave_days_used` can HINCRBY thuong xuyen → Hash hop ly hon. Hash khong serialize nested object tot nhu String JSON.

**Q2**: Redis dang dung 90% memory. Ban se lam gi?

> **Answer**: (1) `INFO memory` xem used_memory, maxmemory, eviction_policy. (2) `redis-cli --bigkeys` tim big key. (3) SCAN + TTL kiem tra key khong co expire. (4) `SLOWLOG GET 10` xem command nang. (5) Xem xet tang maxmemory hoac scale Redis. (6) Review eviction policy co phu hop khong.

**Q3**: Sang ra employee bao check-in bi loi luc 8h. Redis logs thay latency spike. Ban debug the nao?

> **Answer**: (1) `SLOWLOG GET 20` xem command cham. (2) Kiem tra co KEYS * khong. (3) `redis-cli --bigkeys`. (4) `INFO memory` — co gan maxmemory khong? (5) Kiem tra network latency. (6) `INFO stats` → `evicted_keys` xem eviction xay ra khong.

**Q4**: Tai sao khong dung Redis List cho production message queue?

> **Answer**: List khong co: (1) Message acknowledgement — consumer fail thi job mat. (2) Retry mechanism. (3) Dead letter queue. (4) Visibility timeout. (5) Multiple consumer groups. Dung Redis Streams (BullMQ) hoac RabbitMQ/Kafka cho reliable messaging.

**Q5**: Distributed lock SET NX EX — dieu gi xay ra neu process giu lock bi kill sau khi xu ly xong nhung chua DEL?

> **Answer**: Lock tu expire sau EX giay — safety net. Nhung neu job chay lau hon EX → lock expire giua chung → process khac lay lock → race condition. Giai phap: (1) EX du lon cho job. (2) Lock renewal watchdog. (3) Idempotent operation.

**Q6**: Rate limit `/checkin` toi da 3 lan/phut cho moi employee. Implement the nao?

> **Answer**: Fixed window (INCR + EXPIRE): don gian nhung co boundary issue (3 req cuoi phut + 3 req dau phut tiep = 6 trong 2s). Sliding window chinh xac hon voi Sorted Set: ZADD timestamp, ZREMRANGEBYSCORE ngoai window, ZCARD dem. Trade-off: Sorted Set ton memory hon String counter. Voi timekeeping system scale nho (500-2000 employee) → Fixed window du dung, don gian hon.

**Q7**: Tai sao nen dung TTL jitter?

> **Answer**: Neu 2000 employee cung check-in luc 8h, cache duoc set voi EX 600 deu expire luc 8h10. Luc do tat ca request hit DB cung luc → spike. Voi jitter `EX 600 + random(0, 60)` → expire trai ra tu 8h10 den 8h11 → load phan deu.

---

## 9. Appendix: Quick Reference

### Commands Tong Hop

```bash
# String
SET key value [EX sec] [PX ms] [NX] [XX]
GET key | MGET key1 key2 key3
INCR key | INCRBY key n | DECR key | DECRBY key n
GETDEL key | GETEX key EX sec

# Hash
HSET key field value [field value ...]
HGET key field | HMGET key f1 f2 | HGETALL key
HKEYS key | HVALS key | HLEN key
HINCRBY key field n | HINCRBYFLOAT key field f
HEXISTS key field | HDEL key field
HSCAN key cursor [MATCH pat] [COUNT n]

# List
LPUSH key val | RPUSH key val
LPOP key [count] | RPOP key [count]
LLEN key | LRANGE key start stop | LTRIM key start stop
BLPOP key timeout | BRPOP key timeout

# Set
SADD key m1 m2 | SREM key m1 m2
SISMEMBER key m | SMISMEMBER key m1 m2
SCARD key | SMEMBERS key
SUNION k1 k2 | SINTER k1 k2 | SDIFF k1 k2
SSCAN key cursor [MATCH pat] [COUNT n]

# Sorted Set
ZADD key [NX|XX] [GT|LT] score m
ZREM key m | ZPOPMIN key [count] | ZPOPMAX key [count]
ZSCORE key m | ZRANK key m | ZREVRANK key m
ZCARD key | ZCOUNT key min max
ZRANGE key start stop [BYSCORE] [REV] [WITHSCORES] [LIMIT offset count]
ZRANGEBYSCORE key min max [WITHSCORES] [LIMIT offset count]
ZREMRANGEBYSCORE key min max | ZREMRANGEBYRANK key start stop
ZSCAN key cursor [MATCH pat] [COUNT n]

# Key management
TTL key | PTTL key | PERSIST key
EXPIRE key sec | PEXPIRE key ms | EXPIREAT key timestamp
DEL key [key ...] | UNLINK key [key ...]
EXISTS key | TYPE key | OBJECT ENCODING key
SCAN cursor [MATCH pat] [COUNT n] [TYPE type]

# Debug / Monitor
MONITOR
SLOWLOG GET [count] | SLOWLOG RESET | SLOWLOG LEN
MEMORY USAGE key [SAMPLES count]
INFO [section]       # all | memory | keyspace | stats | replication
DBSIZE
CONFIG GET maxmemory
CONFIG SET maxmemory 512mb
```

### Encoding Noi Bo

Redis tu chon encoding toi uu — biet de tranh memory spike:

| Type | Encoding nho | Encoding lon | Nguong chuyen doi |
|---|---|---|---|
| String | embstr | raw | > 44 bytes |
| Hash | listpack | hashtable | > 128 fields hoac value > 64 bytes |
| List | listpack | quicklist | > 128 elements hoac value > 64 bytes |
| Set | listpack / intset | hashtable | > 128 members hoac non-integer |
| Sorted Set | listpack | skiplist | > 128 members hoac value > 64 bytes |

```bash
# Xem encoding hien tai
OBJECT ENCODING cache:employee:100
OBJECT ENCODING leaderboard:early-checkin:2026-07-08
```

**Tai sao can biet?** Khi Hash vuot 128 fields → chuyen hashtable → memory tang dot bien. Biet de thiet ke key split hop ly.

---

*Day 1 hoan thanh. Next: Day 2 — Persistence (RDB/AOF), Replication & High Availability.*
