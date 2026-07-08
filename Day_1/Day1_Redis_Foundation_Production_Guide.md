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

## 1. Lý Thuyết Cốt Lõi

### 1.1 Redis Là Gì?

Redis = **RE**mote **DI**ctionary **S**erver.

Nó là **in-memory data structure store** — không phải "database nhanh". Toàn bộ data sống trong RAM, không cần parse qua disk I/O.

Redis phục vụ các use case:

| Use Case | Lý do dùng Redis |
|---|---|
| Cache | Tránh đọc DB lặp lại |
| Session store | Fast read, TTL tự xóa |
| OTP / token | TTL tự expire |
| Rate limiter | INCR atomic, không race condition |
| Distributed lock | SET NX + EX, atomic |
| Leaderboard | Sorted Set rank by score |
| Counter | INCR/DECR atomic |
| Simple queue | List LPUSH/RPOP |
| Online users | Set |
| Delayed job | Sorted Set + score = timestamp |

> **Rule quan trọng**: Redis **bổ sung** PostgreSQL, không **thay thế**.

---

### 1.2 Redis vs SQL Database

| Tiêu chí | PostgreSQL/MySQL | Redis |
|---|---|---|
| Vị trí trong stack | Source of truth | Cache / Coordination layer |
| Query | SQL phức tạp, JOIN | Lookup theo key |
| Transaction | ACID đầy đủ | Atomic command, MULTI/EXEC giới hạn |
| Data model | Schema, relation | Key-value, data structures |
| Persistence | Durable, WAL | RAM first, persistence optional |
| Scale | Vertical + horizontal (phức tạp) | Horizontal cluster được |
| Latency | ms-tens of ms | sub-ms |
| Memory | Disk unlimited | RAM có giới hạn |

**Câu ghi nhớ**:
> PostgreSQL giữ **tính đúng lâu dài**.
> Redis giữ **tốc độ, trạng thái tạm, coordination nhẹ**.

Timekeeping system: PostgreSQL lưu attendance record chính thức. Redis lưu trạng thái "employee đang online", "counter check-in hôm nay", lock chống duplicate check-in.

---

### 1.3 In-Memory — Tradeoffs Thật Sự

**Ưu điểm:**
- Latency sub-millisecond (so với DB: 5-50ms)
- Throughput cực cao (100k+ ops/sec single instance)
- Data structure phong phú sẵn có

**Nhược điểm (phải nhớ kỹ cho production):**

| Vấn đề | Hậu quả | Giải pháp |
|---|---|---|
| RAM đắt | Không thể lưu tất cả data | Chỉ cache hot data, dùng TTL |
| Restart mất data | Nếu không config persistence | RDB + AOF persistence |
| Memory đầy | Redis từ chối write hoặc evict | Eviction policy phù hợp |
| Out of Memory | Redis crash | Monitor memory, set `maxmemory` |

**Eviction policy cần biết:**

```
noeviction       Từ chối write khi đầy. Nguy hiểm cho cache layer
allkeys-lru      Xóa key ít dùng nhất. Tốt cho cache pure
volatile-lru     Xóa key có TTL ít dùng nhất. Tốt cho mixed workload
volatile-ttl     Xóa key sắp expire sớm nhất
allkeys-random   Xóa random. Tránh dùng
```

**Production timekeeping**: Dùng `allkeys-lru` nếu Redis là pure cache. Dùng `volatile-lru` nếu Redis vừa cache vừa có persistent keys (distributed lock không nên bị evict).

---

### 1.4 Single-Threaded — Ý Nghĩa Thực Tế

Redis xử lý command qua **một main execution thread** duy nhất.

```
Client A: INCR counter:checkin:2026-07-08
Client B: INCR counter:checkin:2026-07-08
```

Kết quả luôn đúng vì Redis xử lý tuần tự. Không bao giờ bị race condition trên single command.

**Hệ quả trong production:**

```
OK  INCR, DECR    Atomic, an toàn
OK  SET NX EX     Atomic lock, không race
OK  ZADD, LPUSH   Atomic
NO  KEYS *        Scan toàn bộ keyspace, block tất cả client khác
NO  Lua script dài  Block event loop
NO  HGETALL trên hash 10k field  Block
```

> **Rule**: Không bao giờ dùng `KEYS *` trong production. Thay bằng `SCAN`.

```bash
# BAD - block toàn bộ trong lúc scan
KEYS cache:employee:*

# GOOD - scan theo batch, không block
SCAN 0 MATCH cache:employee:* COUNT 100
```

**Lưu ý**: Redis 6.0+ có I/O threads xử lý network, nhưng logic xử lý command vẫn single-threaded. Khi nói "single-threaded" = main command processing thread.

---

### 1.5 Key Naming Convention

**Pattern chuẩn**: `{namespace}:{entity}:{identifier}`

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

**Rules quan trọng:**

| Rule | Ví dụ sai | Ví dụ đúng |
|---|---|---|
| Có namespace | `employee:100` | `cache:employee:100` |
| Readable | `e:100:d` | `cache:employee:100` |
| Cache/temp có TTL | `session:abc` không TTL | `session:abc EX 3600` |
| Không dùng tên chung chung | `data`, `temp`, `user` | `cache:employee:100` |
| Entity + identifier | `employee` | `cache:employee:{id}` |

**Hot key problem**: Nếu hàng nghìn request cùng đọc `cache:employee:1` (giám đốc), đây là hot key. Giải pháp: key sharding `cache:employee:1:shard:{0-9}`.

---

### 1.6 TTL / Expiration

```bash
# Set key với TTL
SET cache:employee:100 '{"id":100}' EX 300    # 300 seconds
SET otp:phone:0901234567 "123456" PX 120000   # 120000 milliseconds

# Set TTL sau khi key đã tồn tại
EXPIRE cache:employee:100 300                  # seconds
PEXPIRE cache:employee:100 300000              # milliseconds

# Kiểm tra TTL
TTL cache:employee:100    # 297 (còn 297 giây)
TTL cache:employee:100    # -1  (không có expire)
TTL cache:employee:999    # -2  (key không tồn tại)

# Xóa expire
PERSIST cache:employee:100
```

**Bảng TTL cần nhớ cho timekeeping:**

| Key | TTL | Lý do |
|---|---|---|
| `cache:employee:{id}` | 5-15 phút | Profile thay đổi ít |
| `cache:shift:{id}` | 30-60 phút | Shift config ít thay đổi |
| `session:{token}` | 1-8 giờ | Session expire |
| `otp:phone:{phone}` | 2-5 phút | OTP ngắn |
| `otp:attempt:{phone}` | 15-60 phút | Rate limit window |
| `rate:login:{id}` | 15-60 phút | Sliding/fixed window |
| `lock:checkin:{id}:{date}` | 10-30 giây | Lock ngắn, tự giải phóng |
| `stats:checkin:{date}` | 25 giờ | Hết ngày + buffer |
| `leaderboard:early-checkin:{date}` | 25 giờ | Leaderboard hằng ngày |
| `online:employees` | Không TTL | Managed manually |

---

## 2. Data Types — Chi Tiết Production

### 2.1 String

**Là gì**: Bytes, tối đa 512MB. Binary-safe byte sequence.

**Dùng khi**:
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
INCRBY stats:checkin:2026-07-08 5    # Tăng 5
DECR quota:otp:100                   # Giảm 1

# One-time token: lấy và xóa ngay
GETDEL session:one-time-abc
```

**Pattern — Rate limiter (chú ý pitfall):**

```bash
INCR rate:login:100
EXPIRE rate:login:100 900
# CAUTION: 2 lệnh này KHÔNG atomic -> dùng Lua nếu cần chính xác
```

**Production pitfall**:
- Lưu JSON string OK với cache, nhưng nếu hay update từng field → dùng Hash
- String lưu large JSON (>100KB) → tốn memory, tốn network

---

### 2.2 Hash

**Là gì**: Map key-value bên trong một Redis key. Tương tự một row trong SQL.

**Dùng khi**:
- Object nhiều field, thường update từng field
- Không cần TTL riêng cho từng field
- Tiết kiệm memory nếu field nhỏ (listpack encoding)

```bash
# Lưu employee với nhiều field
HSET employee:100 name "Nguyen Van A" department_id "10" status "active" leave_days_used "2"

# Lấy một field
HGET employee:100 name                     # "Nguyen Van A"

# Lấy tất cả field
HGETALL employee:100

# Lấy nhiều field cụ thể
HMGET employee:100 name department_id

# Tăng số nguyên trong field (atomic)
HINCRBY employee:100 leave_days_used 1     # 3

# Kiểm tra field tồn tại
HEXISTS employee:100 status               # 1

# Xóa field
HDEL employee:100 temp_field

# Đếm số field
HLEN employee:100
```

**Hash vs String (JSON):**

| Tiêu chí | Hash | String (JSON) |
|---|---|---|
| Update từng field thường xuyên | Tốt | Phải đọc-parse-update-write lại |
| Read toàn bộ object | Tốt (HGETALL) | Tốt (GET) |
| TTL cho cả object | EXPIRE | EX |
| Serialize nested object | Khó | JSON.stringify OK |
| Memory (< 128 fields) | Tối ưu listpack | Tương đương |

**Production Pitfall**: `HGETALL` trên hash với hàng nghìn field → Big key, block. Giữ hash < 1000 fields.

---

### 2.3 List

**Là gì**: Doubly-linked list. Push/pop từ cả 2 đầu O(1).

**Dùng khi**:
- Simple queue (FIFO: RPUSH + LPOP)
- Stack (LIFO: LPUSH + LPOP)
- Recent activity / history (giới hạn bằng LTRIM)
- Blocking queue (BRPOP)

```bash
# Queue: producer thêm job
RPUSH queue:emails "job:email:100"
RPUSH queue:emails "job:email:101"

# Queue: consumer lấy theo FIFO
LPOP queue:emails             # "job:email:100"

# Blocking pop: wait tối đa 5 giây
BRPOP queue:emails 5

# Recent activity: giữ 20 hoạt động gần nhất
LPUSH activity:employee:100 "checkin:2026-07-08T08:00:00"
LTRIM activity:employee:100 0 19

# Xem list
LRANGE activity:employee:100 0 -1   # Tất cả
LRANGE activity:employee:100 0 4    # 5 gần nhất
LLEN activity:employee:100
```

**Production Pitfall**:
- List không dedup — cùng một item có thể push nhiều lần
- LRANGE 0 -1 trên list lớn → Big key, block
- **Redis List không phải production-grade message queue** — không có ack, không retry, không dead letter. Dùng BullMQ (Redis Streams) hoặc RabbitMQ/Kafka cho critical work.

---

### 2.4 Set

**Là gì**: Unordered collection, không duplicate, O(1) cho add/remove/membership check.

**Dùng khi**:
- Unique collection
- Membership check (ai đang online?)
- Deduplication
- Tag / permission

```bash
# Track employee đang online
SADD online:employees 100 101 102 103

# Kiểm tra có online không
SISMEMBER online:employees 100    # 1 (có)
SISMEMBER online:employees 999    # 0 (không có)

# Xóa khỏi set
SREM online:employees 100

# Đếm số lượng
SCARD online:employees            # 3

# Lấy tất cả (chỉ dùng khi set nhỏ)
SMEMBERS online:employees         # O(N)

# Set operations
SUNION online:employees vip:employees    # Union
SINTER online:employees admin:employees  # Intersection
SDIFF all:employees online:employees     # Difference (ai offline)
```

**Production Pitfall**:
- `SMEMBERS` trên set lớn → block. Dùng `SSCAN`.
- Set không lưu được metadata (join time, score). Nếu cần → Sorted Set.

---

### 2.5 Sorted Set (ZSet)

**Là gì**: Set với mỗi member kèm một `score` (float). Luôn sorted by score tự động.

**Dùng khi**:
- Leaderboard / ranking
- Priority queue
- Delayed job (score = unix timestamp)
- Time-ordered data
- Rate limit với sliding window chính xác

```bash
# Leaderboard: check-in sớm hôm nay
# Score = unix timestamp (thấp hơn = sớm hơn)
ZADD leaderboard:early-checkin:2026-07-08 1751940000 "employee:100"   # 08:00
ZADD leaderboard:early-checkin:2026-07-08 1751939700 "employee:101"   # 07:55
ZADD leaderboard:early-checkin:2026-07-08 1751940600 "employee:102"   # 08:10

# Top 10 check-in sớm nhất
ZRANGE leaderboard:early-checkin:2026-07-08 0 9 WITHSCORES

# Top 10 check-in muộn nhất (reverse)
ZREVRANGE leaderboard:early-checkin:2026-07-08 0 9 WITHSCORES

# Rank của employee (0-indexed, 0 = sớm nhất)
ZRANK leaderboard:early-checkin:2026-07-08 "employee:101"    # 0 (đứng đầu)

# Score của một employee
ZSCORE leaderboard:early-checkin:2026-07-08 "employee:100"

ZCARD leaderboard:early-checkin:2026-07-08
ZREM leaderboard:early-checkin:2026-07-08 "employee:100"

# Delayed Job Queue
ZADD delayed:jobs 1751944200 "send-report:2026-07-08"
ZRANGEBYSCORE delayed:jobs 0 1751944200 LIMIT 0 10
# Sau khi xử lý: ZREM delayed:jobs "send-report:2026-07-08"

# Sliding Window Rate Limit
ZADD rate:api:100 1751940000000 "req:abc"
ZREMRANGEBYSCORE rate:api:100 0 1751939940000   # xóa request ngoài window 60s
ZCARD rate:api:100                              # đếm request trong window
```

**Production Pitfall**:
- `ZRANGE` / `ZREVRANGE` deprecated từ Redis 6.2. Dùng `ZRANGE ... REV` hoặc `ZRANGE ... BYSCORE`.
- Sorted Set không nên có > 100k members.

---

## 3. Patterns Tổng Hợp

### 3.1 Distributed Lock (Chống Duplicate Check-in)

```bash
# SET NX EX: atomic "set nếu không tồn tại"
SET lock:checkin:100:2026-07-08 "unique-uuid-of-process-a" NX EX 10
# "OK" nếu lock thành công
# nil nếu lock đang bị giữ
```

**Release lock đúng cách** — PHẢI verify value trước khi DEL:

```lua
-- Lua script atomic (check-and-delete)
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
else
    return 0
end
```

**Lý do cần unique value**: Process A set lock, timeout, lock expire, Process B set lock mới. Nếu A DEL không kiểm tra value → xóa nhầm lock của B.

---

### 3.2 Cache-Aside Pattern với TTL Jitter

```typescript
// NestJS service example
async getEmployee(id: number): Promise<Employee> {
  const cacheKey = `cache:employee:${id}`;

  const cached = await this.redis.get(cacheKey);
  if (cached) return JSON.parse(cached);

  const employee = await this.employeeRepo.findOne({ where: { id } });
  if (employee) {
    // TTL jitter: tránh cache stampede
    const ttl = 600 + Math.floor(Math.random() * 60);
    await this.redis.set(cacheKey, JSON.stringify(employee), 'EX', ttl);
  }
  return employee;
}
```

---

### 3.3 Rate Limiter — Atomic với Lua

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
DEL cache:employee:100     # Synchronous: block event loop cho đến khi xóa xong
UNLINK cache:employee:100  # Async: trả lời ngay, xóa ở background thread
```

**Rule production**: Luôn dùng `UNLINK`. Đặc biệt quan trọng với big key.

---

## 4. Bài Tập Day 1

### Bài 1: Thiết kế Key Schema

| Use Case | Key | Data Type | TTL | Lý do |
|---|---|---|---|---|
| Employee profile cache | `cache:employee:{id}` | String (JSON) | 600s | Ít thay đổi |
| Shift config cache | `cache:shift:{id}` | String (JSON) | 1800s | Rất ít thay đổi |
| Attendance summary | `cache:attendance:{emp_id}:{date}` | String (JSON) | 300s | Daily, stale ok |
| Online employees | `online:employees` | Set | No TTL | Managed manually |
| Daily check-in counter | `stats:checkin:{date}` | String (counter) | 25h | Daily reset |
| Early check-in leaderboard | `leaderboard:early-checkin:{date}` | Sorted Set | 25h | Daily |
| OTP code | `otp:phone:{phone}` | String | 120s | Short-lived |
| OTP attempt counter | `otp:attempt:{phone}` | String (counter) | 900s | Rate limit window |
| Login attempts | `rate:login:{user_id}` | String (counter) | 900s | Rate limit |
| Duplicate check-in lock | `lock:checkin:{emp_id}:{date}` | String | 10s | Short lock |
| Recent activity | `activity:employee:{id}` | List | 7d | History |

### Bài 2: Commands Practice (không nhìn tài liệu)

1. Cache employee 100 với TTL 10 phút
2. Đọc employee 100 từ cache
3. Đọc đồng thời employee 100, 101, 102
4. Tăng counter check-in hôm nay (date: 2026-07-08)
5. Set OTP `456789` cho phone `0901234567` expire 2 phút
6. Set employee 200 vào Hash với field: name, department_id, status
7. Tăng leave_days_used của employee 200 thêm 1
8. Thêm employee 100 vào `online:employees`
9. Kiểm tra employee 999 có online không
10. Add leaderboard hôm nay: employee 100 check-in lúc 08:05 (unix: 1751940300)
11. Lấy top 5 check-in sớm nhất hôm nay
12. Scan keys prefix `cache:employee:` mà không block

---

## 5. Mini Lab

### Lab Setup

```bash
# Chạy Redis local với Docker
docker run -d --name redis-lab -p 6379:6379 redis:7

# Kết nối
docker exec -it redis-lab redis-cli

# Monitor real-time commands (terminal khác)
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
TTL cache:employee:100    # -2 (key không tồn tại)
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

# Top 5 sớm nhất
ZRANGE leaderboard:early-checkin:2026-07-08 0 4 WITHSCORES

# Rank (0-indexed, 0 = sớm nhất)
ZRANK leaderboard:early-checkin:2026-07-08 "employee:100"     # 2
ZRANK leaderboard:early-checkin:2026-07-08 "employee:103"     # 0 (sớm nhất)
```

### Lab 5: Distributed Lock Simulation

```bash
# Process A: lock
SET lock:checkin:100:2026-07-08 "process-a-uuid" NX EX 10
# OK

# Process B: có lock cùng key
SET lock:checkin:100:2026-07-08 "process-b-uuid" NX EX 10
# nil (thất bại)

# Process A: verify rồi release
GET lock:checkin:100:2026-07-08    # "process-a-uuid"
DEL lock:checkin:100:2026-07-08    # 1
```

### Lab 6: KEYS vs SCAN

```bash
# Tạo 1000 keys test
EVAL "for i=1,1000 do redis.call('SET', 'cache:employee:'..i, 'data', 'EX', 600) end" 0

# BAD - production blocked
KEYS cache:employee:*

# GOOD - cursor-based, an toàn
SCAN 0 MATCH cache:employee:* COUNT 100
# Lặp với cursor trả về cho đến khi cursor = 0

INFO keyspace
```

---

## 6. Checklist Day 1

### Kiến thức

- [ ] Giải thích Redis là gì và dùng cho gì (không đọc note)
- [ ] Nêu 3 tradeoff của in-memory
- [ ] Giải thích single-threaded và hệ quả thực tế
- [ ] Thiết kế key name cho 5 use case bất kỳ
- [ ] Giải thích TTL -1 và TTL -2
- [ ] Nêu data type nào dùng cho leaderboard và tại sao
- [ ] Giải thích tại sao KEYS * bị cấm

### Thực hành

- [ ] Chạy Redis local bằng Docker
- [ ] CRUD với String, Hash, List, Set, Sorted Set
- [ ] Set và verify TTL
- [ ] Chạy SCAN thay KEYS *
- [ ] Simulate distributed lock với SET NX EX
- [ ] Dùng MONITOR để xem real-time commands

---

## 7. Production Pitfalls

### P1: KEYS * trong Production

```bash
# Tuyệt đối không dùng
KEYS *
KEYS cache:*

# Luôn dùng SCAN
SCAN 0 MATCH cache:employee:* COUNT 100
```

**Hậu quả thực tế**: KEYS * trên Redis 1 triệu key có thể block 200-500ms. Tất cả request khác bị treo. Production down.

---

### P2: Big Key

Big key = value rất lớn, hoặc collection có quá nhiều member.

```bash
# Detect big key
redis-cli --bigkeys

# Memory của key cụ thể
MEMORY USAGE cache:employee:100
```

**Rule**:
- String: < 1MB OK, > 10MB cần review
- Hash/List/Set/ZSet: < 10,000 members OK

---

### P3: Cache Key Không Có TTL

```bash
# Memory leak
SET cache:employee:100 '{"id":100}'

# Đúng
SET cache:employee:100 '{"id":100}' EX 600
```

**Hậu quả**: Memory tăng mãi, eviction kick in, cache stale vĩnh viễn.

---

### P4: Cache Stampede (Thundering Herd)

**Khi nào**: Cache expire đồng loạt của popular key → tất cả request hit DB → DB quá tải.

**Giải pháp**:
- TTL jitter: `EX (base + random(0, 60))` — tránh expire đồng loạt
- Probabilistic early refresh: refresh trước khi expire
- Lock pattern: chỉ 1 request rebuild cache, các request khác dùng stale hoặc đợi

---

### P5: Race Condition trong Rate Limiter

```bash
# INCR + EXPIRE không atomic
INCR rate:login:100
EXPIRE rate:login:100 900
# Nếu INCR thành công nhưng process crash → key không có TTL → memory leak
```

**Giải pháp**: Lua script atomic (xem Section 3.3).

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

**Vấn đề**: Job chạy 30s nhưng lock chỉ 10s → lock expire giữa chừng → race condition.

```typescript
// Watchdog: renew lock mỗi 5s
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

**Q1**: Bạn cache employee profile bằng String JSON. Team leader hỏi: "Sao không dùng Hash?"

> **Answer**: String tốt khi read toàn bộ object một lần, ít update individual field. Hash tốt khi thường xuyên HINCRBY hoặc HSET từng field. Profile thường chỉ đọc → String simpler, ít overhead. Nếu có field như `leave_days_used` cần HINCRBY thường xuyên → Hash hợp lý hơn. Hash không serialize nested object tốt như String JSON.

**Q2**: Redis đang dùng 90% memory. Bạn sẽ làm gì?

> **Answer**: (1) `INFO memory` xem used_memory, maxmemory, eviction_policy. (2) `redis-cli --bigkeys` tìm big key. (3) SCAN + TTL kiểm tra key không có expire. (4) `SLOWLOG GET 10` xem command nặng. (5) Xem xét tăng maxmemory hoặc scale Redis. (6) Review eviction policy có phù hợp không.

**Q3**: Sáng ra employee báo check-in bị lỗi lúc 8h. Redis logs thấy latency spike. Bạn debug thế nào?

> **Answer**: (1) `SLOWLOG GET 20` xem command chậm. (2) Kiểm tra có KEYS * không. (3) `redis-cli --bigkeys`. (4) `INFO memory` — có gần maxmemory không? (5) Kiểm tra network latency. (6) `INFO stats` → `evicted_keys` xem eviction xảy ra không.

**Q4**: Tại sao không dùng Redis List cho production message queue?

> **Answer**: List không có: (1) Message acknowledgement — consumer fail thì job mất. (2) Retry mechanism. (3) Dead letter queue. (4) Visibility timeout. (5) Multiple consumer groups. Dùng Redis Streams (BullMQ) hoặc RabbitMQ/Kafka cho reliable messaging.

**Q5**: Distributed lock SET NX EX — điều gì xảy ra nếu process giữ lock bị kill sau khi xử lý xong nhưng chưa DEL?

> **Answer**: Lock tự expire sau EX giây — safety net. Nhưng nếu job chạy lâu hơn EX → lock expire giữa chừng → process khác lấy lock → race condition. Giải pháp: (1) EX đủ lớn cho job. (2) Lock renewal watchdog. (3) Idempotent operation.

**Q6**: Rate limit `/checkin` tối đa 3 lần/phút cho mỗi employee. Implement thế nào?

> **Answer**: Fixed window (INCR + EXPIRE): đơn giản nhưng có boundary issue (3 req cuối phút + 3 req đầu phút tiếp = 6 trong 2s). Sliding window chính xác hơn với Sorted Set: ZADD timestamp, ZREMRANGEBYSCORE ngoài window, ZCARD đếm. Trade-off: Sorted Set tốn memory hơn String counter. Với timekeeping system scale nhỏ (500-2000 employee) → Fixed window đủ dùng, đơn giản hơn.

**Q7**: Tại sao nên dùng TTL jitter?

> **Answer**: Nếu 2000 employee cùng check-in lúc 8h, cache được set với EX 600 đều expire lúc 8h10. Lúc đó tất cả request hit DB cùng lúc → spike. Với jitter `EX 600 + random(0, 60)` → expire trải ra từ 8h10 đến 8h11 → load phân đều.

---

## 9. Appendix: Quick Reference

### Commands Tổng Hợp

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

### Encoding Nội Bộ

Redis tự chọn encoding tối ưu — biết để tránh memory spike:

| Type | Encoding nhỏ | Encoding lớn | Ngưỡng chuyển đổi |
|---|---|---|---|
| String | embstr | raw | > 44 bytes |
| Hash | listpack | hashtable | > 128 fields hoặc value > 64 bytes |
| List | listpack | quicklist | > 128 elements hoặc value > 64 bytes |
| Set | listpack / intset | hashtable | > 128 members hoặc non-integer |
| Sorted Set | listpack | skiplist | > 128 members hoặc value > 64 bytes |

```bash
# Xem encoding hiện tại
OBJECT ENCODING cache:employee:100
OBJECT ENCODING leaderboard:early-checkin:2026-07-08
```

**Tại sao cần biết?** Khi Hash vượt 128 fields → chuyển hashtable → memory tăng đột biến. Biết để thiết kế key split hợp lý.

---

*Day 1 hoàn thành. Next: Day 2 — Persistence (RDB/AOF), Replication & High Availability.*
