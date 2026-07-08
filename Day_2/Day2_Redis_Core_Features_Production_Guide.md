# Day 2: Redis Core Features — Production Guide

> **Mindset**: Day 2 la khi ban hieu Redis khong chi la key-value store. Cac tinh nang nay la diem phan biet giua nguoi biet Redis co ban va nguoi hieu Redis that su.

---

## Muc Tieu

Sau Day 2, ban phai tra loi duoc **khong can suy nghi**:

- [ ] Bitmap dung khi nao, khac Set o dau?
- [ ] HyperLogLog la gi, tai sao khong dung Set de dem unique?
- [ ] Pub/Sub mat message khi consumer offline — dung hay sai?
- [ ] Streams khac Pub/Sub o dieu gi?
- [ ] MULTI/EXEC co giong SQL transaction khong?
- [ ] WATCH dung de lam gi?
- [ ] Lua script co gi hon Pipeline?
- [ ] Pipeline co atomic khong?
- [ ] RDB vs AOF — chon cai nao cho production?

**Context Project**: Timekeeping System — NestJS backend, PostgreSQL source of truth. Scale: ~500-2000 employees.

---

## Keywords Day 2

```
Redis Bitmap SETBIT GETBIT BITCOUNT BITOP
Redis HyperLogLog PFADD PFCOUNT PFMERGE
Redis Pub/Sub PUBLISH SUBSCRIBE UNSUBSCRIBE
Redis Streams XADD XREAD XGROUP XREADGROUP XACK
Redis Streams XPENDING XCLAIM XAUTOCLAIM
Redis consumer group pending entries
Redis MULTI EXEC DISCARD WATCH
Redis Lua EVAL EVALSHA SCRIPT LOAD
Redis Pipeline multi-exec vs pipeline
Redis RDB snapshot AOF append-only
Redis persistence durability
```

---

## 1. Bitmap

### 1.1 La Gi?

Bitmap la String duoc dung nhu mot mang bit. Moi bit co the SETBIT/GETBIT theo offset.

```
Key: attendance:2026-07-08
Offset 0   = employee ID 0
Offset 100 = employee ID 100
Offset 999 = employee ID 999
```

Toi da 2^32 bit = 512MB = 4 billion employee IDs — trong thuc te, 2000 employees chi dung **2000 bits = 250 bytes**.

### 1.2 Use Case

| Use Case | Key Pattern | Offset |
|---|---|---|
| Daily attendance flag | `attendance:{date}` | employee_id |
| Monthly attendance | `attendance:monthly:{emp_id}:{year_month}` | ngay trong thang (1-31) |
| Feature flag per user | `feature:new-ui:{date}` | user_id |
| Daily active users | `dau:{date}` | user_id |

### 1.3 Commands

```bash
# Danh dau employee 100 da check-in ngay 2026-07-08
SETBIT attendance:2026-07-08 100 1
# Ket qua: 0 (gia tri cu cua bit do)

# Danh dau them
SETBIT attendance:2026-07-08 101 1
SETBIT attendance:2026-07-08 102 1
SETBIT attendance:2026-07-08 50 1

# Kiem tra employee 100 da check-in chua
GETBIT attendance:2026-07-08 100    # 1 (da check-in)
GETBIT attendance:2026-07-08 999    # 0 (chua check-in)

# Dem tong so nguoi da check-in (dem so bit = 1)
BITCOUNT attendance:2026-07-08         # 4

# Dem trong khoang byte (range la byte, khong phai bit)
BITCOUNT attendance:2026-07-08 0 -1   # Tat ca

# BITPOS: tim bit dau tien co gia tri 0 hoac 1
BITPOS attendance:2026-07-08 1        # Tim bit 1 dau tien (employee check-in som nhat theo ID)
BITPOS attendance:2026-07-08 0        # Tim bit 0 dau tien (employee chua check-in co ID nho nhat)

# BITOP: phep toan bitwise giua nhieu key
BITOP AND result:both attendance:2026-07-07 attendance:2026-07-08
# result:both = employees da check-in CA HAI ngay

BITOP OR result:either attendance:2026-07-07 attendance:2026-07-08
# result:either = employees da check-in IT NHAT MOT trong hai ngay

BITOP NOT result:absent attendance:2026-07-08
# result:absent = employees CHUA check-in (bit flip)
```

### 1.4 Bitmap vs Set — Khi Nao Dung Cai Nao?

| Tieu chi | Bitmap | Set |
|---|---|---|
| Member type | Integer offset | Any string |
| Memory | Rat tiet kiem (bit-level) | Lon hon (string overhead) |
| Membership check | O(1) GETBIT | O(1) SISMEMBER |
| Count members | O(n) BITCOUNT | O(1) SCARD |
| Set operations | BITOP AND/OR/XOR | SINTER/SUNION/SDIFF |
| Lay danh sach member | Kho (phai scan tung bit) | SMEMBERS |
| Range offset lon | Gap ton memory | Khong bi anh huong |

**Rule**:
- Member ID la so nguyen, so luong lon, can tiet kiem memory → **Bitmap**
- Can lay danh sach member, member la string → **Set**

**Caution**: Neu employee ID lon (vd: UUID hoac ID = 10_000_000) → Bitmap ton memory vi se allocate het cac byte trung gian. Bitmap toi uu nhat khi IDs dense va tu 0.

### 1.5 Production Pitfalls

**P1: Sparse Bitmap Gap Memory**
```bash
# Employee ID = 1,000,000 → Bitmap phai allocate ~125KB chi cho 1 bit
SETBIT attendance:2026-07-08 1000000 1
MEMORY USAGE attendance:2026-07-08    # ~125KB chi cho 1 entry
```
Giai phap: Neu ID khong lien tuc, dung Set hoac map ID thanh dense index.

**P2: BITCOUNT la O(n)**
```bash
# BITCOUNT tren 1MB bitmap → O(n) theo byte length
# Tuong doi nhanh nhung van block neu bitmap rat lon
BITCOUNT huge-bitmap    # Caution tren bitmap > 10MB
```

---

## 2. HyperLogLog

### 2.1 La Gi?

HyperLogLog (HLL) la probabilistic data structure de **dem unique elements** voi sai so xap xi **0.81%**, dung **toi da 12KB memory** bat ke co bao nhieu phan tu.

So sanh voi Set:

| Tieu chi | Set | HyperLogLog |
|---|---|---|
| Memory 1M unique | ~50-100MB | ~12KB |
| Chinh xac | 100% exact | ~99.19% (+-0.81%) |
| Lay danh sach | SMEMBERS | Khong the |
| Union | SUNIONSTORE | PFMERGE |
| Use case | Can exact, can danh sach | Chi can count, tiet kiem memory |

### 2.2 Use Case

| Use Case | Key Pattern |
|---|---|
| Unique active employees theo ngay | `hll:active-emp:{date}` |
| Unique visitors dashboard | `hll:dashboard-visitors:{date}` |
| Unique IPs per endpoint | `hll:ip:{endpoint}:{date}` |
| Unique features used | `hll:feature:{feature_name}:{month}` |

### 2.3 Commands

```bash
# Add unique employees duoc het ngay
PFADD hll:active-emp:2026-07-08 emp100
PFADD hll:active-emp:2026-07-08 emp101 emp102 emp103
PFADD hll:active-emp:2026-07-08 emp100   # Trung lap → khong tang count

# Dem so luong unique (xap xi)
PFCOUNT hll:active-emp:2026-07-08        # ~3

# Merge nhieu HLL lai de dem union
PFADD hll:active-emp:2026-07-07 emp100 emp200 emp300
PFMERGE hll:active-emp:week hll:active-emp:2026-07-07 hll:active-emp:2026-07-08
PFCOUNT hll:active-emp:week             # ~5 (emp100, emp101, emp102, emp103, emp200, emp300)
```

### 2.4 Khi Nao Dung HyperLogLog?

**Dung HLL khi**:
- Chi can biet **bao nhieu** unique (khong can biet LA NHUNG AI)
- Volume lon (trieu+ unique)
- Chap nhan sai so nho (~0.81%)
- Memory la uu tien

**Khong dung HLL khi**:
- Can biet chinh xac 100%
- Can lay danh sach member
- Volume nho → dung Set binh thuong

**Timekeeping context**: HLL hop ly cho "bao nhieu employee unique da mo dashboard hom nay". Khong hop ly cho "ai da check-in" (phai biet danh sach cu the).

### 2.5 Production Pitfall

**HLL khong merge duoc voi Set/ZSet** — la data structure rieng biet. Khong the lay ra danh sach. Mot khi da chon HLL, khong lay lai duoc members.

---

## 3. Pub/Sub

### 3.1 La Gi?

Pub/Sub la messaging model **fire-and-forget**: publisher gui message toi channel, tat ca subscriber dang listen deu nhan. Khong co persistence.

```
Publisher → [channel: attendance-events] → Subscriber A
                                          → Subscriber B
                                          → Subscriber C
```

### 3.2 Commands

```bash
# --- Terminal 1: Subscribe ---
SUBSCRIBE attendance-events
# Blocking — nhan het message gui toi channel nay

# Subscribe nhieu channel
SUBSCRIBE attendance-events shift-events

# Pattern subscribe (wildcard)
PSUBSCRIBE attendance-*    # Nhan tat ca channel bat dau bang "attendance-"

# --- Terminal 2: Publish ---
PUBLISH attendance-events '{"employee_id":100,"type":"check_in","timestamp":"2026-07-08T08:00:00"}'
# Ket qua: so luong subscriber da nhan

PUBLISH attendance-events '{"employee_id":101,"type":"check_out","timestamp":"2026-07-08T17:00:00"}'

# Unsubscribe
UNSUBSCRIBE attendance-events
PUNSUBSCRIBE attendance-*
```

### 3.3 Pub/Sub Use Case trong Timekeeping

```
Employee check-in
     ↓
NestJS API: PUBLISH attendance-events {json}
     ↓
Subscriber 1: Real-time dashboard update (WebSocket)
Subscriber 2: Notification service
Subscriber 3: Log aggregator
```

### 3.4 Pub/Sub Limitations — QUAN TRONG

| Diem yeu | Hau qua | Giai phap |
|---|---|---|
| Consumer offline → mat message | Notification bi mat | Dung Streams |
| Khong co persistence | Redis restart → mat tat ca message dang pending | Dung Streams |
| Khong co ack | Khong biet consumer da xu ly xong chua | Dung Streams |
| Khong co retry | Consumer fail → message mat | Dung Streams |
| Slow consumer block nhanh | Redis buffer day → ngat ket noi | Streams hoac buffer consumer |

**Rule**: Pub/Sub chi tot cho **realtime notification nhe, mat duoc**. Neu can reliable → **Streams**.

### 3.5 Pub/Sub vs Streams vs List

| Tieu chi | Pub/Sub | Streams | List (BLPOP) |
|---|---|---|---|
| Durability | Khong | Co | Co |
| Ack | Khong | Co | Khong |
| Consumer group | Khong | Co | Khong |
| Multiple consumers | Broadcast | Fan-out theo group | Compete consume |
| Replay history | Khong | Co (XREAD from ID) | Khong |
| Pending/retry | Khong | Co (XPENDING/XCLAIM) | Khong |
| Use case | Live notification | Reliable event processing | Simple queue |

---

## 4. Streams

### 4.1 La Gi?

Redis Streams la **append-only log** voi:
- Message ID co thu tu (timestamp-sequence)
- Consumer groups (multiple consumers chia nhau work)
- Message acknowledgement
- Pending entries (chua duoc ack)
- Retry / claim (lay lai message chua xu ly)

Tuong tu Kafka nhung nhe hon, native trong Redis.

### 4.2 Stream ID

Moi message co ID dinh dang: `{milliseconds}-{sequence}`

```
1751940000000-0    # Timestamp ms = 2026-07-08 08:00:00, sequence = 0
1751940000000-1    # Cung timestamp, sequence tiep theo
*                  # Redis tu sinh ID theo thoi gian thuc
```

### 4.3 Commands — Core

```bash
# XADD: Them message vao stream
XADD stream:attendance-events * employee_id 100 type check_in timestamp 2026-07-08T08:00:00
# * = tu dong sinh ID
# Ket qua: "1751940000000-0"

XADD stream:attendance-events * employee_id 101 type check_in timestamp 2026-07-08T08:05:00
# Ket qua: "1751940300000-0"

# XLEN: Dem so message trong stream
XLEN stream:attendance-events    # 2

# XRANGE: Doc message tu ID bat dau den ID ket thuc
XRANGE stream:attendance-events - +               # Tat ca message
XRANGE stream:attendance-events - + COUNT 10      # 10 message dau
XRANGE stream:attendance-events 1751940000000-0 + # Tu ID nay tro di

# XREVRANGE: Doc nguoc
XREVRANGE stream:attendance-events + - COUNT 5   # 5 message moi nhat

# XREAD: Doc message tu nhieu stream
XREAD COUNT 10 STREAMS stream:attendance-events 0
# 0 = doc tu dau
# $ = chi lay message moi sau thoi diem nay

# XREAD blocking (consumer pattern)
XREAD COUNT 10 BLOCK 5000 STREAMS stream:attendance-events $
# Doi toi da 5 giay cho message moi
```

### 4.4 Consumer Groups

Consumer group cho phep nhieu worker chia nhau xu ly message (fan-out theo cong viec, khong broadcast).

```bash
# Tao consumer group
XGROUP CREATE stream:attendance-events workers $ MKSTREAM
# workers = ten group
# $ = chi doc message moi tu bay gio (dung 0 de doc tu dau)
# MKSTREAM = tao stream neu chua co

# --- Worker 1 lay message ---
XREADGROUP GROUP workers worker-1 COUNT 5 STREAMS stream:attendance-events >
# > = lay message chua duoc deliver cho bat ky consumer nao trong group

# --- Worker 2 lay message khac ---
XREADGROUP GROUP workers worker-2 COUNT 5 STREAMS stream:attendance-events >

# --- Worker 1 ack sau khi xu ly xong ---
XACK stream:attendance-events workers 1751940000000-0
# Neu khong XACK, message nam trong pending list

# Xem pending messages (chua duoc ack)
XPENDING stream:attendance-events workers - + 10
# Hien thi: message ID, consumer dang giu, thoi gian giu, so lan deliver

# XCLAIM: Lay lai message cua consumer chet/timeout
XCLAIM stream:attendance-events workers worker-2 30000 1751940000000-0
# 30000ms = message nay da nam pending > 30s → claim lai

# XAUTOCLAIM: Tu dong claim nhieu message pending
XAUTOCLAIM stream:attendance-events workers worker-2 30000 0-0 COUNT 10
```

### 4.5 Streams Pattern cho Timekeeping

```
Employee check-in API
        ↓
XADD stream:attendance-events * emp_id 100 type check_in

        ↓ [consumer group: workers]
        
Worker 1: doc message, luu vao PostgreSQL, XACK
Worker 2: doc message, gui notification, XACK
Worker 3: doc message, update dashboard cache, XACK
```

**Dead Letter Queue pattern**:
```bash
# Monitor: message bi deliver > 3 lan ma van chua ack
XPENDING stream:attendance-events workers - + 100

# Neu deliver_count > 3 → move sang DLQ
XADD stream:attendance-events:dlq * original_id {id} reason "max_retries_exceeded"
XACK stream:attendance-events workers {id}
```

### 4.6 Giu Stream Khong Lon Vo Han

```bash
# MAXLEN: Gioi han so message trong stream
XADD stream:attendance-events MAXLEN ~ 10000 * emp_id 100 type check_in
# ~ = approximate trim (nhanh hon exact MAXLEN)
# Giu toi da 10000 message moi nhat

# Trim thu cong
XTRIM stream:attendance-events MAXLEN ~ 10000
```

---

## 5. Transactions: MULTI/EXEC/WATCH

### 5.1 MULTI/EXEC

```bash
MULTI              # Bat dau transaction — moi command sau do bi queue
INCR stats:checkin:2026-07-08
EXPIRE stats:checkin:2026-07-08 86400
HSET summary:2026-07-08 total 1 late 0
EXEC               # Thuc thi tat ca command da queue
```

**Ket qua EXEC**: Tra ve array cac ket qua cua tung command.

**DISCARD**: Huy bo transaction, xoa queue.

```bash
MULTI
INCR stats:checkin:2026-07-08
DISCARD            # Xoa het, khong thuc thi gi
```

### 5.2 Redis Transaction Khac SQL Transaction The Nao?

| Tieu chi | SQL Transaction | Redis MULTI/EXEC |
|---|---|---|
| Rollback khi command loi | Co (ROLLBACK) | Khong — command khac van chay |
| Rollback khi runtime error | Co | Khong — command loi bi skip, command khac chay tiep |
| Syntax error | Rollback tat ca | Discard tat ca (biet ngay) |
| Isolation | ACID isolation | Khong isolation — command khac van vao duoc |
| Atomicity | Day du | Chi atomicity theo nghia "tat ca hay khong co gi" cho syntax, khong phai logic |

```bash
MULTI
INCR counter:1         # OK
LPUSH counter:1 "foo"  # Runtime error (sai type) nhung EXEC van chay
INCR counter:2         # Van duoc thuc thi du lenh truoc loi
EXEC
# Ket qua: [1, ERROR, 1]
# counter:2 da tang — khong co rollback!
```

**Ket luan**: Redis MULTI/EXEC dam bao **nhom command chay khong bi xen giua**, nhung **khong co rollback logic** nhu SQL.

### 5.3 WATCH — Optimistic Locking

WATCH cho phep detect neu key bi thay doi truoc khi EXEC.

```bash
# Pattern: Tang quota cua employee neu con du

WATCH quota:employee:100
current = GET quota:employee:100    # 5

# Neu quota:employee:100 bi thay doi boi client khac sau WATCH va truoc EXEC...
MULTI
DECR quota:employee:100
EXEC
# → nil (abort) neu quota:employee:100 da bi thay doi
# → [4] neu thanh cong (khong ai chen vao)
```

**Ung dung thuc te**:
```typescript
// NestJS example: WATCH pattern
async decrementQuota(empId: number): Promise<boolean> {
  const key = `quota:employee:${empId}`;
  
  let result = null;
  do {
    await this.redis.watch(key);
    const current = parseInt(await this.redis.get(key) || '0');
    
    if (current <= 0) {
      await this.redis.unwatch();
      return false; // Khong du quota
    }
    
    const multi = this.redis.multi();
    multi.decr(key);
    result = await multi.exec(); // null neu WATCH detect thay doi
  } while (result === null); // Retry neu bi abort
  
  return true;
}
```

---

## 6. Lua Scripting

### 6.1 Tai Sao Dung Lua?

Lua cho phep nhieu Redis command chay **atomic** trong mot cuoc goi. Trong khi:
- Pipeline: Gom nhieu command nhung KHONG atomic
- MULTI/EXEC: Atomic nhung KHONG co dieu kien (if/else)
- Lua: Atomic VA co dieu kien/logic

### 6.2 Syntax Co Ban

```bash
# EVAL "script" numkeys key1 key2 ... arg1 arg2 ...
EVAL "return redis.call('GET', KEYS[1])" 1 cache:employee:100
# 1 = so luong KEYS
# KEYS[1] = cache:employee:100
# Ket qua: gia tri cua key

# Nhieu command
EVAL "
  local val = redis.call('GET', KEYS[1])
  if val then
    redis.call('EXPIRE', KEYS[1], ARGV[1])
    return val
  end
  return nil
" 1 cache:employee:100 600
```

### 6.3 Cac Pattern Pho Bien

**Pattern 1: Release Distributed Lock An Toan**
```lua
-- Check value truoc khi DEL (tranh xoa lock cua process khac)
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
else
    return 0
end
```

```bash
EVAL "if redis.call('GET', KEYS[1]) == ARGV[1] then return redis.call('DEL', KEYS[1]) else return 0 end" 1 lock:checkin:100:2026-07-08 "process-a-uuid"
```

**Pattern 2: Atomic Rate Limiter**
```lua
-- Increment va set TTL trong mot buoc
local current = redis.call('GET', KEYS[1])
if current == false then
    redis.call('SET', KEYS[1], 1, 'EX', ARGV[1])
    return 1
else
    local new_val = redis.call('INCR', KEYS[1])
    return new_val
end
```

```bash
EVAL "local c = redis.call('GET', KEYS[1]) if c == false then redis.call('SET', KEYS[1], 1, 'EX', ARGV[1]) return 1 else return redis.call('INCR', KEYS[1]) end" 1 rate:login:100 900
```

**Pattern 3: Atomic Quota Check-and-Decrement**
```lua
-- Kiem tra quota du truoc khi giam
local quota = tonumber(redis.call('GET', KEYS[1]))
if quota == nil or quota <= 0 then
    return 0  -- Khong du quota
end
redis.call('DECR', KEYS[1])
return 1  -- Thanh cong
```

**Pattern 4: Cache Getset voi Conditional TTL**
```lua
-- Lay gia tri, refresh TTL neu con song
local val = redis.call('GET', KEYS[1])
if val then
    local ttl = tonumber(redis.call('TTL', KEYS[1]))
    if ttl < tonumber(ARGV[1]) then
        redis.call('EXPIRE', KEYS[1], ARGV[2])
    end
    return val
end
return false
```

### 6.4 EVALSHA — Cache Script

```bash
# Load script len Redis, nhan SHA
SCRIPT LOAD "if redis.call('GET', KEYS[1]) == ARGV[1] then return redis.call('DEL', KEYS[1]) else return 0 end"
# Ket qua: "3c6ef1aa7408a42dc97d9dc47c28d15d3069f4d7"

# Goi bang SHA (nhanh hon, it bandwidth hon)
EVALSHA 3c6ef1aa7408a42dc97d9dc47c28d15d3069f4d7 1 lock:checkin:100:2026-07-08 "process-a-uuid"

# Kiem tra script ton tai
SCRIPT EXISTS 3c6ef1aa7408a42dc97d9dc47c28d15d3069f4d7
```

### 6.5 Lua Pitfalls

| Pitfall | Hau qua | Giai phap |
|---|---|---|
| Lua script chay qua lau | Block event loop | Giu script ngan, khong loop lon |
| Script loi giua chung | Command truoc do van co hieu luc (khong rollback) | Test ky truoc deploy |
| Ket qua Lua va Redis type | Lua nil khac Redis nil | Kiem tra ky type conversion |
| KEYS[] bat dau tu 1, khong phai 0 | Index error | KEYS[1], KEYS[2], ... |

---

## 7. Pipeline

### 7.1 La Gi?

Pipeline cho phep gui nhieu command len Redis trong mot network request, nhan tat ca ket qua mot lan.

```
Khong Pipeline:                  Pipeline:
Client → COMMAND 1 → Server      Client → [CMD1, CMD2, CMD3] → Server
Client ← response 1 ← Server     Client ← [res1, res2, res3] ← Server
Client → COMMAND 2 → Server
Client ← response 2 ← Server
Client → COMMAND 3 → Server
Client ← response 3 ← Server
```

**Network round-trip**: 3 commands = 3 RTTs → voi Pipeline = 1 RTT.

### 7.2 Pipeline KHONG Atomic

```
Khong Pipeline + MULTI/EXEC:        Pipeline:
Toan bo atomic                      Moi command van rieng le
Khong command nao chen vao duoc     Command khac co the chen vao giua
```

Pipeline chi giam **network latency**. Khong bao dam **atomicity**.

### 7.3 Use Cases

```typescript
// NestJS: Warm cache cho 10 employees
async warmEmployeeCache(ids: number[]): Promise<void> {
  const employees = await this.employeeRepo.find({
    where: { id: In(ids) }
  });

  const pipeline = this.redis.pipeline();
  for (const emp of employees) {
    const ttl = 600 + Math.floor(Math.random() * 60); // TTL jitter
    pipeline.set(`cache:employee:${emp.id}`, JSON.stringify(emp), 'EX', ttl);
  }
  await pipeline.exec();
}

// Bulk get: doc cache cua nhieu employee
async bulkGetEmployees(ids: number[]): Promise<(Employee | null)[]> {
  const pipeline = this.redis.pipeline();
  for (const id of ids) {
    pipeline.get(`cache:employee:${id}`);
  }
  const results = await pipeline.exec();
  return results.map(([err, val]) => {
    if (err || !val) return null;
    return JSON.parse(val as string);
  });
}
```

### 7.4 Pipeline vs MULTI/EXEC vs Lua

| Tieu chi | Pipeline | MULTI/EXEC | Lua |
|---|---|---|---|
| Giam network RTT | Co | Co (1 round trip) | Co (1 round trip) |
| Atomic | Khong | Co | Co |
| Dieu kien (if/else) | Khong | Khong | Co |
| Block server | It hon | It hon | Co the (neu script lon) |
| Use case | Bulk get/set, batch | Nhom command khong logic | Complex atomic logic |

**Decision tree**:
- Chi can giam network → **Pipeline**
- Can nhom atomic, khong logic → **MULTI/EXEC**
- Can atomic + logic → **Lua**

### 7.5 Pipeline Pitfalls

| Pitfall | Hau qua | Giai phap |
|---|---|---|
| Pipeline qua lon | Memory spike o client va server | Batch theo nhom 100-500 |
| Lam nham pipeline voi atomic | Race condition | Dung MULTI/EXEC neu can atomic |
| Error handling trong pipeline | Mot command loi khong ngung cai khac | Kiem tra ket qua tung command |

```typescript
// Batch pipeline theo chunk de tranh memory spike
async bulkSet(items: {key: string, value: string, ttl: number}[]): Promise<void> {
  const CHUNK_SIZE = 200;
  for (let i = 0; i < items.length; i += CHUNK_SIZE) {
    const chunk = items.slice(i, i + CHUNK_SIZE);
    const pipeline = this.redis.pipeline();
    chunk.forEach(item => pipeline.set(item.key, item.value, 'EX', item.ttl));
    await pipeline.exec();
  }
}
```

---

## 8. Persistence — Gioi Thieu

Day 4 se hoc sau. Day 2 hieu concept de biet luc configure.

### 8.1 RDB (Redis Database Snapshot)

RDB = Periodic snapshot cua toan bo data, luu thanh binary file `.rdb`.

```
Redis RAM → [BGSAVE moi 15 phut] → /data/dump.rdb (binary)
```

**Dac diem**:
- File compact, load nhanh khi restart
- Co the mat data giua 2 lan snapshot
- It anh huong performance (BGSAVE dung fork)

**Cau hinh mac dinh**:
```
save 3600 1     # Neu >= 1 key thay doi trong 1 gio → snapshot
save 300 100    # Neu >= 100 key thay doi trong 5 phut → snapshot
save 60 10000   # Neu >= 10000 key thay doi trong 1 phut → snapshot
```

### 8.2 AOF (Append-Only File)

AOF = Ghi moi write command vao file log.

```
Redis RAM → [ghi moi command] → /data/appendonly.aof (text/binary)
```

**Dac diem**:
- It mat data hon RDB (co the chi mat 1 second)
- File lon hon RDB
- Co the rewrite/compact de thu nho (BGREWRITEAOF)

**Fsync options**:
```
appendfsync always      # Flush moi command → an toan nhat, cham nhat
appendfsync everysec    # Flush moi giay → balance tot (mat toi da 1 giay data)
appendfsync no          # OS quyet dinh → nhanh nhat, mat nhieu nhat
```

### 8.3 No Persistence — Pure Cache

```
# redis.conf
save ""           # Tat RDB
appendonly no     # Tat AOF
```

Restart → mat tat ca. Dung cho pure cache layer, data lay lai tu PostgreSQL duoc.

### 8.4 Nen Chon Gi?

| Use case | Persistence mode |
|---|---|
| Pure cache (data lay lai duoc) | No persistence |
| Cache + can recovery nhanh | RDB |
| Session, lock, counter (mat it OK) | AOF everysec |
| Khong muon mat data | AOF always hoac RDB + AOF |

**Timekeeping system**:
- Session, OTP, lock → `AOF everysec` (mat toi da 1 giay OK)
- Employee cache → No persistence (lay lai tu PostgreSQL khi restart)

---

## 9. Bai Tap Day 2

### Bai 1: Bitmap Attendance

```bash
# Setup: Danh dau attendance cho ngay 2026-07-08
SETBIT attendance:2026-07-08 50 1
SETBIT attendance:2026-07-08 100 1
SETBIT attendance:2026-07-08 101 1
SETBIT attendance:2026-07-08 102 1
SETBIT attendance:2026-07-08 200 1

# Cau hoi:
# 1. Employee 100 da check-in chua?
# 2. Tong co bao nhieu nguoi check-in?
# 3. Employee nao co ID nho nhat ma chua check-in?
# 4. Employee nao da check-in CA HAI ngay 07 va 08?

SETBIT attendance:2026-07-07 100 1
SETBIT attendance:2026-07-07 200 1
SETBIT attendance:2026-07-07 300 1
BITOP AND result:both attendance:2026-07-07 attendance:2026-07-08
BITCOUNT result:both
```

### Bai 2: HyperLogLog Dashboard

```bash
# Simulate employees dang mo dashboard
PFADD hll:dashboard:2026-07-08 emp100 emp101 emp102
PFADD hll:dashboard:2026-07-08 emp100   # Trung lap
PFADD hll:dashboard:2026-07-08 emp103 emp104

PFCOUNT hll:dashboard:2026-07-08        # Bao nhieu unique?

# Merge voi ngay hom truoc
PFADD hll:dashboard:2026-07-07 emp100 emp200 emp300
PFMERGE hll:dashboard:week hll:dashboard:2026-07-07 hll:dashboard:2026-07-08
PFCOUNT hll:dashboard:week
```

### Bai 3: Pub/Sub Check-in Event

```bash
# Terminal 1: Subscribe
SUBSCRIBE attendance-events

# Terminal 2: Publish check-in events
PUBLISH attendance-events '{"emp_id":100,"type":"check_in","time":"08:00"}'
PUBLISH attendance-events '{"emp_id":101,"type":"check_in","time":"08:05"}'

# Quan sat Terminal 1 nhan message khong?

# Thu: Dong Terminal 1, publish them message o Terminal 2
# Quan sat: message bi mat
```

### Bai 4: Streams Consumer Group

```bash
# Tao stream va consumer group
XGROUP CREATE stream:attendance-events workers $ MKSTREAM

# Add message
XADD stream:attendance-events * emp_id 100 type check_in
XADD stream:attendance-events * emp_id 101 type check_in
XADD stream:attendance-events * emp_id 102 type check_out

# Worker 1 doc va xu ly
XREADGROUP GROUP workers worker-1 COUNT 5 STREAMS stream:attendance-events >

# Xem pending (chua ack)
XPENDING stream:attendance-events workers - + 10

# Ack sau khi xu ly
XACK stream:attendance-events workers {message-id-from-above}

# Xem pending con lai
XPENDING stream:attendance-events workers - + 10
```

### Bai 5: Transaction MULTI/EXEC

```bash
# Atomic: tang counter va set TTL
MULTI
INCR stats:checkin:2026-07-08
EXPIRE stats:checkin:2026-07-08 86400
EXEC

# Quan sat: ca 2 command chay hay chi 1?
# Thu loi: MULTI, LPUSH stats:checkin:2026-07-08 "foo", INCR stats:checkin:2026-07-08, EXEC
# Quan sat: Redis xu ly loi the nao?
```

### Bai 6: Lua Script Release Lock

```bash
# Setup lock
SET lock:checkin:100:2026-07-08 "my-uuid-abc" NX EX 30

# Release lock dung cach voi Lua
EVAL "if redis.call('GET', KEYS[1]) == ARGV[1] then return redis.call('DEL', KEYS[1]) else return 0 end" 1 lock:checkin:100:2026-07-08 "my-uuid-abc"
# Ket qua: 1 (thanh cong)

# Thu voi sai token
SET lock:checkin:101:2026-07-08 "correct-uuid" NX EX 30
EVAL "if redis.call('GET', KEYS[1]) == ARGV[1] then return redis.call('DEL', KEYS[1]) else return 0 end" 1 lock:checkin:101:2026-07-08 "wrong-uuid"
# Ket qua: 0 (khong del, vi token sai)
```

### Bai 7: Pipeline Bulk Cache

```bash
# Dung redis-cli pipeline mode
# Tao file commands.txt:
SET cache:employee:1 '{"id":1,"name":"Emp 1"}' EX 600
SET cache:employee:2 '{"id":2,"name":"Emp 2"}' EX 600
SET cache:employee:3 '{"id":3,"name":"Emp 3"}' EX 600
GET cache:employee:1
GET cache:employee:2
GET cache:employee:3

# Chay pipeline
cat commands.txt | docker exec -i redis-lab redis-cli --pipe

# Hoac trong redis-cli, dung MULTI/EXEC de minh hoa
MULTI
GET cache:employee:1
GET cache:employee:2
GET cache:employee:3
EXEC
```

---

## 10. Checklist Day 2

### Kien thuc

- [ ] Giai thich Bitmap dung cho boolean flag, va khi nao chon Bitmap vs Set
- [ ] Giai thich HyperLogLog la approximate unique count, khong lay duoc danh sach
- [ ] Giai thich Pub/Sub mat message khi consumer offline — tai sao
- [ ] So sanh Streams vs Pub/Sub: ack, consumer group, replay
- [ ] Giai thich MULTI/EXEC khac SQL transaction the nao (khong co rollback logic)
- [ ] Giai thich WATCH la optimistic locking
- [ ] Giai thich Lua atomic hon Pipeline
- [ ] Giai thich Pipeline chi giam RTT, khong atomic
- [ ] Neu duoc 3 persistence mode va khi nao dung moi mode

### Thuc hanh

- [ ] SETBIT/GETBIT/BITCOUNT cho attendance flag
- [ ] BITOP AND cho intersection 2 ngay
- [ ] PFADD/PFCOUNT/PFMERGE cho HyperLogLog
- [ ] Subscribe va publish trong 2 terminal rieng
- [ ] XADD/XGROUP/XREADGROUP/XACK
- [ ] MULTI/EXEC voi check-in counter
- [ ] WATCH pattern
- [ ] Lua release lock
- [ ] Pipeline bulk get/set

---

## 11. Production Pitfalls Day 2

### P1: Pub/Sub Consumer Gap → Mat Message

```
T1: Subscriber dang chay, nhan message OK
T2: Subscriber restart (deploy moi, crash)
T3: Publisher gui 50 messages
T4: Subscriber chay lai — 50 messages da mat
```

**Giai phap**: Dung Streams voi consumer group. XREADGROUP + XACK dam bao delivery.

---

### P2: MULTI/EXEC Khong Co Rollback

```bash
MULTI
INCR budget:department:10    # Giam budget
INCR purchase:order:500      # Tang don hang
EXEC
```

Neu command thu 2 loi runtime (sai type), command thu 1 da chay. Khong co rollback.

**Giai phap**: Voi logic phuc tap can rollback → dung Lua (co the tra ve error, goi pcode xu ly), hoac don gian hoa business logic.

---

### P3: Lua Script Dai Block Event Loop

```lua
-- Lua script voi loop lon → NGUY HIEM
for i=1,100000 do
  redis.call('INCR', 'counter:'..i)
end
```

Redis block trong thoi gian nay. Tat ca client khac phai doi.

**Giai phap**: Giu Lua script ngan (< 1ms). Neu can xu ly nhieu → chia nho, hoac xu ly o application layer.

---

### P4: Pipeline Qua Lon

```typescript
// BAD: Pipeline 100,000 commands
const pipeline = redis.pipeline();
for (let i = 0; i < 100000; i++) {
  pipeline.set(`key:${i}`, 'value');
}
await pipeline.exec();   // Memory spike o ca client va server
```

**Giai phap**: Chunk pipeline theo batch 200-500 commands.

---

### P5: Streams Pending List Phong To

Neu consumer khong XACK, pending list tang vo han.

```bash
# Kiem tra pending entries
XPENDING stream:attendance-events workers - + 100
XLEN stream:attendance-events

# Monitor XPENDING dinh ky
```

**Giai phap**: Monitor pending list trong alert system. Implement XCLAIM/XAUTOCLAIM cho message bi stale.

---

### P6: Bitmap Dense vs Sparse

```bash
# BAD: ID gap → ton memory
SETBIT attendance:2026-07-08 1000000 1    # Allocate 125KB cho 1 bit
SETBIT attendance:2026-07-08 5000000 1    # Allocate 625KB

# GOOD: Map ID thanh sequential index neu ID khong dense
# employee_offset = employee_id % max_employees hoac dung map
```

---

## 12. Senior Review Questions Day 2

**Q1**: Khi nao chon Bitmap va khi nao chon Set de track daily attendance?

> **Answer**: Bitmap tot khi employee ID la integer, so luong lon (>10k), can tiet kiem memory, va can BITOP (AND/OR) de tinh intersection/union giua cac ngay. Set tot khi ID la string, so luong nho, hoac can lay danh sach ai da check-in (SMEMBERS). Caution: Bitmap voi ID lon hoac sparse lam mat loi the memory.

**Q2**: Ban can dem unique employees da mo dashboard trong thang. Dung HLL hay Set?

> **Answer**: HLL neu scale lon (>100k unique) va chi can count, khong can biet danh sach ai. Set neu can exact count, can biet cu the ai, hoac scale nho (< 10k unique). Voi timekeeping 2000 employee, Set la du — khong dang cau phuc tap HLL.

**Q3**: Tai sao Pub/Sub khong phu hop cho attendance event processing?

> **Answer**: Consumer offline → mat message (khong buffer). Khong co ack → khong biet consumer xu ly thanh cong chua. Khong co retry khi consumer fail. Khong co consumer group de scale. Dung Streams voi XREADGROUP + XACK. Pub/Sub chi dung cho notification nhe (realtime dashboard update) co the mat duoc.

**Q4**: Mo ta WATCH va ung dung cu the trong timekeeping.

> **Answer**: WATCH implement optimistic locking — neu key bi thay doi sau WATCH va truoc EXEC, EXEC tra ve nil (abort). Ung dung: decrement OTP quota cua employee. WATCH quota:emp:100, GET current value, MULTI + DECR + EXEC. Neu employee dung 2 device cung luc, mot trong hai request se fail va phai retry. An toan hon Lua cho use case don gian.

**Q5**: Giai thich Pipeline vs MULTI/EXEC vs Lua cho batch cache set.

> **Answer**: Pipeline: Gui tat ca SET cung luc, 1 RTT thay vi N RTT. Khong atomic — OK cho batch cache set vi tung cache entry la doc lap. MULTI/EXEC: Atomic nhom SET, nhung overhead cua MULTI/EXEC khong can thiet cho cache set doc lap. Lua: Atomic voi logic phuc tap. Over-engineering cho simple batch set. Ket luan: Dung Pipeline cho batch cache operations thong thuong.

**Q6**: Streams consumer group — dieu gi xay ra khi worker-1 crash sau khi XREADGROUP nhung truoc XACK?

> **Answer**: Message nam trong pending list, XPENDING cho thay worker-1 dang giu message do. Nếu co heartbeat/monitor, worker-2 co the XCLAIM message sau timeout (vd: 30s). Hoac XAUTOCLAIM tu dong claim. Neu khong co, message bi stuck → implement monitor va auto-claim la mandatory trong production.

---

## Appendix: Quick Reference Day 2

### Bitmap

```bash
SETBIT key offset 0|1
GETBIT key offset
BITCOUNT key [start end]
BITPOS key 0|1 [start [end]]
BITOP AND|OR|XOR|NOT destkey key [key ...]
```

### HyperLogLog

```bash
PFADD key element [element ...]    # Return 1 neu HLL changed, 0 neu khong
PFCOUNT key [key ...]              # Approximate unique count
PFMERGE destkey sourcekey [sourcekey ...]
```

### Pub/Sub

```bash
PUBLISH channel message
SUBSCRIBE channel [channel ...]
UNSUBSCRIBE [channel ...]
PSUBSCRIBE pattern [pattern ...]   # Pattern: attendance-*
PUNSUBSCRIBE [pattern ...]
PUBSUB CHANNELS [pattern]          # List active channels
PUBSUB NUMSUB [channel ...]        # Subscriber count per channel
```

### Streams

```bash
XADD key [MAXLEN [~] count] * field value [field value ...]
XLEN key
XRANGE key - + [COUNT n]
XREVRANGE key + - [COUNT n]
XREAD [COUNT n] [BLOCK ms] STREAMS key [key ...] id [id ...]
XGROUP CREATE key group id|$ [MKSTREAM]
XREADGROUP GROUP group consumer [COUNT n] [BLOCK ms] STREAMS key [key ...] >
XACK key group id [id ...]
XPENDING key group [[IDLE ms] - + count [consumer]]
XCLAIM key group consumer min-idle-ms id [id ...]
XAUTOCLAIM key group consumer min-idle-ms start [COUNT count]
XTRIM key MAXLEN [~] count
XDEL key id [id ...]
XINFO STREAM key
XINFO GROUPS key
XINFO CONSUMERS key group
```

### Transaction

```bash
MULTI
EXEC
DISCARD
WATCH key [key ...]
UNWATCH
```

### Lua

```bash
EVAL script numkeys [key [key ...]] [arg [arg ...]]
EVALSHA sha1 numkeys [key [key ...]] [arg [arg ...]]
SCRIPT LOAD script
SCRIPT EXISTS sha1 [sha1 ...]
SCRIPT FLUSH [ASYNC|SYNC]
```

---

*Day 2 hoan thanh. Next: Day 3 — Redis Cluster, Sentinel & High Availability.*
