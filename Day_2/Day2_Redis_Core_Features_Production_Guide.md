# Day 2: Redis Core Features — Production Guide

> **Mindset**: Day 2 là khi bạn hiểu Redis không chỉ là key-value store. Các tính năng này là điểm phân biệt giữa người biết Redis cơ bản và người hiểu Redis thật sự.

---

## Mục Tiêu

Sau Day 2, bạn phải trả lời được **không cần suy nghĩ**:

- [ ] Bitmap dùng khi nào, khác Set ở đâu?
- [ ] HyperLogLog là gì, tại sao không dùng Set để đếm unique?
- [ ] Pub/Sub mất message khi consumer offline — đúng hay sai?
- [ ] Streams khác Pub/Sub ở điều gì?
- [ ] MULTI/EXEC có giống SQL transaction không?
- [ ] WATCH dùng để làm gì?
- [ ] Lua script có gì hơn Pipeline?
- [ ] Pipeline có atomic không?
- [ ] RDB vs AOF — chọn cái nào cho production?

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

### 1.1 Là Gì?

Bitmap là String được dùng như một mảng bit. Mỗi bit có thể SETBIT/GETBIT theo offset.

```
Key: attendance:2026-07-08
Offset 0   = employee ID 0
Offset 100 = employee ID 100
Offset 999 = employee ID 999
```

Tối đa 2^32 bit = 512MB = 4 billion employee IDs — trong thực tế, 2000 employees chỉ dùng **2000 bits = 250 bytes**.

### 1.2 Use Case

| Use Case | Key Pattern | Offset |
|---|---|---|
| Daily attendance flag | `attendance:{date}` | employee_id |
| Monthly attendance | `attendance:monthly:{emp_id}:{year_month}` | ngày trong tháng (1-31) |
| Feature flag per user | `feature:new-ui:{date}` | user_id |
| Daily active users | `dau:{date}` | user_id |

### 1.3 Commands

```bash
# Đánh dấu employee 100 đã check-in ngày 2026-07-08
SETBIT attendance:2026-07-08 100 1
# Kết quả: 0 (giá trị cũ của bit đó)

# Đánh dấu thêm
SETBIT attendance:2026-07-08 101 1
SETBIT attendance:2026-07-08 102 1
SETBIT attendance:2026-07-08 50 1

# Kiểm tra employee 100 đã check-in chưa
GETBIT attendance:2026-07-08 100    # 1 (đã check-in)
GETBIT attendance:2026-07-08 999    # 0 (chưa check-in)

# Đếm tổng số người đã check-in (đếm số bit = 1)
BITCOUNT attendance:2026-07-08         # 4

# Đếm trong khoảng byte (range là byte, không phải bit)
BITCOUNT attendance:2026-07-08 0 -1   # Tất cả

# BITPOS: tìm bit đầu tiên có giá trị 0 hoặc 1
BITPOS attendance:2026-07-08 1        # Tìm bit 1 đầu tiên (employee check-in sớm nhất theo ID)
BITPOS attendance:2026-07-08 0        # Tìm bit 0 đầu tiên (employee chưa check-in có ID nhỏ nhất)

# BITOP: phép toán bitwise giữa nhiều key
BITOP AND result:both attendance:2026-07-07 attendance:2026-07-08
# result:both = employees đã check-in CẢ HAI ngày

BITOP OR result:either attendance:2026-07-07 attendance:2026-07-08
# result:either = employees đã check-in ÍT NHẤT MỘT trong hai ngày

BITOP NOT result:absent attendance:2026-07-08
# result:absent = employees CHƯA check-in (bit flip)
```

### 1.4 Bitmap vs Set — Khi Nào Dùng Cái Nào?

| Tiêu chí | Bitmap | Set |
|---|---|---|
| Member type | Integer offset | Any string |
| Memory | Rất tiết kiệm (bit-level) | Lớn hơn (string overhead) |
| Membership check | O(1) GETBIT | O(1) SISMEMBER |
| Count members | O(n) BITCOUNT | O(1) SCARD |
| Set operations | BITOP AND/OR/XOR | SINTER/SUNION/SDIFF |
| Lấy danh sách member | Khó (phải scan từng bit) | SMEMBERS |
| Range offset lớn | Gặp tốn memory | Không bị ảnh hưởng |

**Rule**:
- Member ID là số nguyên, số lượng lớn, cần tiết kiệm memory → **Bitmap**
- Cần lấy danh sách member, member là string → **Set**

**Caution**: Nếu employee ID lớn (vd: UUID hoặc ID = 10_000_000) → Bitmap tốn memory vì sẽ allocate hết các byte trung gian. Bitmap tối ưu nhất khi IDs dense và từ 0.

### 1.5 Production Pitfalls

**P1: Sparse Bitmap Gặp Memory**
```bash
# Employee ID = 1,000,000 → Bitmap phải allocate ~125KB chỉ cho 1 bit
SETBIT attendance:2026-07-08 1000000 1
MEMORY USAGE attendance:2026-07-08    # ~125KB chỉ cho 1 entry
```
Giải pháp: Nếu ID không liên tục, dùng Set hoặc map ID thành dense index.

**P2: BITCOUNT là O(n)**
```bash
# BITCOUNT trên 1MB bitmap → O(n) theo byte length
# Tương đối nhanh nhưng vẫn block nếu bitmap rất lớn
BITCOUNT huge-bitmap    # Caution trên bitmap > 10MB
```

---

## 2. HyperLogLog

### 2.1 Là Gì?

HyperLogLog (HLL) là probabilistic data structure để **đếm unique elements** với sai số xấp xỉ **0.81%**, dùng **tối đa 12KB memory** bất kể có bao nhiêu phần tử.

So sánh với Set:

| Tiêu chí | Set | HyperLogLog |
|---|---|---|
| Memory 1M unique | ~50-100MB | ~12KB |
| Chính xác | 100% exact | ~99.19% (+-0.81%) |
| Lấy danh sách | SMEMBERS | Không thể |
| Union | SUNIONSTORE | PFMERGE |
| Use case | Cần exact, cần danh sách | Chỉ cần count, tiết kiệm memory |

### 2.2 Use Case

| Use Case | Key Pattern |
|---|---|
| Unique active employees theo ngày | `hll:active-emp:{date}` |
| Unique visitors dashboard | `hll:dashboard-visitors:{date}` |
| Unique IPs per endpoint | `hll:ip:{endpoint}:{date}` |
| Unique features used | `hll:feature:{feature_name}:{month}` |

### 2.3 Commands

```bash
# Add unique employees được hết ngày
PFADD hll:active-emp:2026-07-08 emp100
PFADD hll:active-emp:2026-07-08 emp101 emp102 emp103
PFADD hll:active-emp:2026-07-08 emp100   # Trùng lặp → không tăng count

# Đếm số lượng unique (xấp xỉ)
PFCOUNT hll:active-emp:2026-07-08        # ~3

# Merge nhiều HLL lại để đếm union
PFADD hll:active-emp:2026-07-07 emp100 emp200 emp300
PFMERGE hll:active-emp:week hll:active-emp:2026-07-07 hll:active-emp:2026-07-08
PFCOUNT hll:active-emp:week             # ~5 (emp100, emp101, emp102, emp103, emp200, emp300)
```

### 2.4 Khi Nào Dùng HyperLogLog?

**Dùng HLL khi**:
- Chỉ cần biết **bao nhiêu** unique (không cần biết LÀ NHỮNG AI)
- Volume lớn (triệu+ unique)
- Chấp nhận sai số nhỏ (~0.81%)
- Memory là ưu tiên

**Không dùng HLL khi**:
- Cần biết chính xác 100%
- Cần lấy danh sách member
- Volume nhỏ → dùng Set bình thường

**Timekeeping context**: HLL hợp lý cho "bao nhiêu employee unique đã mở dashboard hôm nay". Không hợp lý cho "ai đã check-in" (phải biết danh sách cụ thể).

### 2.5 Production Pitfall

**HLL không merge được với Set/ZSet** — là data structure riêng biệt. Không thể lấy ra danh sách. Một khi đã chọn HLL, không lấy lại được members.

---

## 3. Pub/Sub

### 3.1 Là Gì?

Pub/Sub là messaging model **fire-and-forget**: publisher gửi message tới channel, tất cả subscriber đang listen đều nhận. Không có persistence.

```
Publisher → [channel: attendance-events] → Subscriber A
                                          → Subscriber B
                                          → Subscriber C
```

### 3.2 Commands

```bash
# --- Terminal 1: Subscribe ---
SUBSCRIBE attendance-events
# Blocking — nhận hết message gửi tới channel này

# Subscribe nhiều channel
SUBSCRIBE attendance-events shift-events

# Pattern subscribe (wildcard)
PSUBSCRIBE attendance-*    # Nhận tất cả channel bắt đầu bằng "attendance-"

# --- Terminal 2: Publish ---
PUBLISH attendance-events '{"employee_id":100,"type":"check_in","timestamp":"2026-07-08T08:00:00"}'
# Kết quả: số lượng subscriber đã nhận

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

### 3.4 Pub/Sub Limitations — QUAN TRỌNG

| Điểm yếu | Hậu quả | Giải pháp |
|---|---|---|
| Consumer offline → mất message | Notification bị mất | Dùng Streams |
| Không có persistence | Redis restart → mất tất cả message đang pending | Dùng Streams |
| Không có ack | Không biết consumer đã xử lý xong chưa | Dùng Streams |
| Không có retry | Consumer fail → message mất | Dùng Streams |
| Slow consumer block nhanh | Redis buffer đầy → ngắt kết nối | Streams hoặc buffer consumer |

**Rule**: Pub/Sub chỉ tốt cho **realtime notification nhẹ, mất được**. Nếu cần reliable → **Streams**.

### 3.5 Pub/Sub vs Streams vs List

| Tiêu chí | Pub/Sub | Streams | List (BLPOP) |
|---|---|---|---|
| Durability | Không | Có | Có |
| Ack | Không | Có | Không |
| Consumer group | Không | Có | Không |
| Multiple consumers | Broadcast | Fan-out theo group | Compete consume |
| Replay history | Không | Có (XREAD từ ID) | Không |
| Pending/retry | Không | Có (XPENDING/XCLAIM) | Không |
| Use case | Live notification | Reliable event processing | Simple queue |

---

## 4. Streams

### 4.1 Là Gì?

Redis Streams là **append-only log** với:
- Message ID có thứ tự (timestamp-sequence)
- Consumer groups (multiple consumers chia nhau work)
- Message acknowledgement
- Pending entries (chưa được ack)
- Retry / claim (lấy lại message chưa xử lý)

Tương tự Kafka nhưng nhẹ hơn, native trong Redis.

### 4.2 Stream ID

Mỗi message có ID định dạng: `{milliseconds}-{sequence}`

```
1751940000000-0    # Timestamp ms = 2026-07-08 08:00:00, sequence = 0
1751940000000-1    # Cùng timestamp, sequence tiếp theo
*                  # Redis tự sinh ID theo thời gian thực
```

### 4.3 Commands — Core

```bash
# XADD: Thêm message vào stream
XADD stream:attendance-events * employee_id 100 type check_in timestamp 2026-07-08T08:00:00
# * = tự động sinh ID
# Kết quả: "1751940000000-0"

XADD stream:attendance-events * employee_id 101 type check_in timestamp 2026-07-08T08:05:00
# Kết quả: "1751940300000-0"

# XLEN: Đếm số message trong stream
XLEN stream:attendance-events    # 2

# XRANGE: Đọc message từ ID bắt đầu đến ID kết thúc
XRANGE stream:attendance-events - +               # Tất cả message
XRANGE stream:attendance-events - + COUNT 10      # 10 message đầu
XRANGE stream:attendance-events 1751940000000-0 + # Từ ID này trở đi

# XREVRANGE: Đọc ngược
XREVRANGE stream:attendance-events + - COUNT 5   # 5 message mới nhất

# XREAD: Đọc message từ nhiều stream
XREAD COUNT 10 STREAMS stream:attendance-events 0
# 0 = đọc từ đầu
# $ = chỉ lấy message mới sau thời điểm này

# XREAD blocking (consumer pattern)
XREAD COUNT 10 BLOCK 5000 STREAMS stream:attendance-events $
# Đợi tối đa 5 giây cho message mới
```

### 4.4 Consumer Groups

Consumer group cho phép nhiều worker chia nhau xử lý message (fan-out theo công việc, không broadcast).

```bash
# Tạo consumer group
XGROUP CREATE stream:attendance-events workers $ MKSTREAM
# workers = tên group
# $ = chỉ đọc message mới từ bây giờ (dùng 0 để đọc từ đầu)
# MKSTREAM = tạo stream nếu chưa có

# --- Worker 1 lấy message ---
XREADGROUP GROUP workers worker-1 COUNT 5 STREAMS stream:attendance-events >
# > = lấy message chưa được deliver cho bất kỳ consumer nào trong group

# --- Worker 2 lấy message khác ---
XREADGROUP GROUP workers worker-2 COUNT 5 STREAMS stream:attendance-events >

# --- Worker 1 ack sau khi xử lý xong ---
XACK stream:attendance-events workers 1751940000000-0
# Nếu không XACK, message nằm trong pending list

# Xem pending messages (chưa được ack)
XPENDING stream:attendance-events workers - + 10
# Hiển thị: message ID, consumer đang giữ, thời gian giữ, số lần deliver

# XCLAIM: Lấy lại message của consumer chết/timeout
XCLAIM stream:attendance-events workers worker-2 30000 1751940000000-0
# 30000ms = message này đã nằm pending > 30s → claim lại

# XAUTOCLAIM: Tự động claim nhiều message pending
XAUTOCLAIM stream:attendance-events workers worker-2 30000 0-0 COUNT 10
```

### 4.5 Streams Pattern cho Timekeeping

```
Employee check-in API
        ↓
XADD stream:attendance-events * emp_id 100 type check_in

        ↓ [consumer group: workers]
        
Worker 1: đọc message, lưu vào PostgreSQL, XACK
Worker 2: đọc message, gửi notification, XACK
Worker 3: đọc message, update dashboard cache, XACK
```

**Dead Letter Queue pattern**:
```bash
# Monitor: message bị deliver > 3 lần mà vẫn chưa ack
XPENDING stream:attendance-events workers - + 100

# Nếu deliver_count > 3 → move sang DLQ
XADD stream:attendance-events:dlq * original_id {id} reason "max_retries_exceeded"
XACK stream:attendance-events workers {id}
```

### 4.6 Giữ Stream Không Lớn Vô Hạn

```bash
# MAXLEN: Giới hạn số message trong stream
XADD stream:attendance-events MAXLEN ~ 10000 * emp_id 100 type check_in
# ~ = approximate trim (nhanh hơn exact MAXLEN)
# Giữ tối đa 10000 message mới nhất

# Trim thủ công
XTRIM stream:attendance-events MAXLEN ~ 10000
```

---

## 5. Transactions: MULTI/EXEC/WATCH

### 5.1 MULTI/EXEC

```bash
MULTI              # Bắt đầu transaction — mỗi command sau đó bị queue
INCR stats:checkin:2026-07-08
EXPIRE stats:checkin:2026-07-08 86400
HSET summary:2026-07-08 total 1 late 0
EXEC               # Thực thi tất cả command đã queue
```

**Kết quả EXEC**: Trả về array các kết quả của từng command.

**DISCARD**: Hủy bỏ transaction, xóa queue.

```bash
MULTI
INCR stats:checkin:2026-07-08
DISCARD            # Xóa hết, không thực thi gì
```

### 5.2 Redis Transaction Khác SQL Transaction Thế Nào?

| Tiêu chí | SQL Transaction | Redis MULTI/EXEC |
|---|---|---|
| Rollback khi command lỗi | Có (ROLLBACK) | Không — command khác vẫn chạy |
| Rollback khi runtime error | Có | Không — command lỗi bị skip, command khác chạy tiếp |
| Syntax error | Rollback tất cả | Discard tất cả (biết ngay) |
| Isolation | ACID isolation | Không isolation — command khác vẫn vào được |
| Atomicity | Đầy đủ | Chỉ atomicity theo nghĩa "tất cả hay không có gì" cho syntax, không phải logic |

```bash
MULTI
INCR counter:1         # OK
LPUSH counter:1 "foo"  # Runtime error (sai type) nhưng EXEC vẫn chạy
INCR counter:2         # Vẫn được thực thi dù lệnh trước lỗi
EXEC
# Kết quả: [1, ERROR, 1]
# counter:2 đã tăng — không có rollback!
```

**Kết luận**: Redis MULTI/EXEC đảm bảo **nhóm command chạy không bị xen giữa**, nhưng **không có rollback logic** như SQL.

### 5.3 WATCH — Optimistic Locking

WATCH cho phép detect nếu key bị thay đổi trước khi EXEC.

```bash
# Pattern: Tăng quota của employee nếu còn đủ

WATCH quota:employee:100
current = GET quota:employee:100    # 5

# Nếu quota:employee:100 bị thay đổi bởi client khác sau WATCH và trước EXEC...
MULTI
DECR quota:employee:100
EXEC
# → nil (abort) nếu quota:employee:100 đã bị thay đổi
# → [4] nếu thành công (không ai chen vào)
```

**Ứng dụng thực tế**:
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
      return false; // Không đủ quota
    }
    
    const multi = this.redis.multi();
    multi.decr(key);
    result = await multi.exec(); // null nếu WATCH detect thay đổi
  } while (result === null); // Retry nếu bị abort
  
  return true;
}
```

---

## 6. Lua Scripting

### 6.1 Tại Sao Dùng Lua?

Lua cho phép nhiều Redis command chạy **atomic** trong một cuộc gọi. Trong khi:
- Pipeline: Gom nhiều command nhưng KHÔNG atomic
- MULTI/EXEC: Atomic nhưng KHÔNG có điều kiện (if/else)
- Lua: Atomic VÀ có điều kiện/logic

### 6.2 Syntax Cơ Bản

```bash
# EVAL "script" numkeys key1 key2 ... arg1 arg2 ...
EVAL "return redis.call('GET', KEYS[1])" 1 cache:employee:100
# 1 = số lượng KEYS
# KEYS[1] = cache:employee:100
# Kết quả: giá trị của key

# Nhiều command
EVAL "
  local val = redis.call('GET', KEYS[1])
  if val then
    redis.call('EXPIRE', KEYS[1], ARGV[1])
    return val
  end
  return nil
" 1 cache:employee:100 600
```

### 6.3 Các Pattern Phổ Biến

**Pattern 1: Release Distributed Lock An Toàn**
```lua
-- Check value trước khi DEL (tránh xóa lock của process khác)
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
-- Increment và set TTL trong một bước
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
-- Kiểm tra quota đủ trước khi giảm
local quota = tonumber(redis.call('GET', KEYS[1]))
if quota == nil or quota <= 0 then
    return 0  -- Không đủ quota
end
redis.call('DECR', KEYS[1])
return 1  -- Thành công
```

**Pattern 4: Cache Getset với Conditional TTL**
```lua
-- Lấy giá trị, refresh TTL nếu còn sống
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
# Load script lên Redis, nhận SHA
SCRIPT LOAD "if redis.call('GET', KEYS[1]) == ARGV[1] then return redis.call('DEL', KEYS[1]) else return 0 end"
# Kết quả: "3c6ef1aa7408a42dc97d9dc47c28d15d3069f4d7"

# Gọi bằng SHA (nhanh hơn, ít bandwidth hơn)
EVALSHA 3c6ef1aa7408a42dc97d9dc47c28d15d3069f4d7 1 lock:checkin:100:2026-07-08 "process-a-uuid"

# Kiểm tra script tồn tại
SCRIPT EXISTS 3c6ef1aa7408a42dc97d9dc47c28d15d3069f4d7
```

### 6.5 Lua Pitfalls

| Pitfall | Hậu quả | Giải pháp |
|---|---|---|
| Lua script chạy quá lâu | Block event loop | Giữ script ngắn, không loop lớn |
| Script lỗi giữa chừng | Command trước đó vẫn có hiệu lực (không rollback) | Test kỹ trước deploy |
| Kết quả Lua và Redis type | Lua nil khác Redis nil | Kiểm tra kỹ type conversion |
| KEYS[] bắt đầu từ 1, không phải 0 | Index error | KEYS[1], KEYS[2], ... |

---

## 7. Pipeline

### 7.1 Là Gì?

Pipeline cho phép gửi nhiều command lên Redis trong một network request, nhận tất cả kết quả một lần.

```
Không Pipeline:                  Pipeline:
Client → COMMAND 1 → Server      Client → [CMD1, CMD2, CMD3] → Server
Client ← response 1 ← Server     Client ← [res1, res2, res3] ← Server
Client → COMMAND 2 → Server
Client ← response 2 ← Server
Client → COMMAND 3 → Server
Client ← response 3 ← Server
```

**Network round-trip**: 3 commands = 3 RTTs → với Pipeline = 1 RTT.

### 7.2 Pipeline KHÔNG Atomic

```
Không Pipeline + MULTI/EXEC:        Pipeline:
Toàn bộ atomic                      Mỗi command vẫn riêng lẻ
Không command nào chen vào được     Command khác có thể chen vào giữa
```

Pipeline chỉ giảm **network latency**. Không bảo đảm **atomicity**.

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

// Bulk get: đọc cache của nhiều employee
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

| Tiêu chí | Pipeline | MULTI/EXEC | Lua |
|---|---|---|---|
| Giảm network RTT | Có | Có (1 round trip) | Có (1 round trip) |
| Atomic | Không | Có | Có |
| Điều kiện (if/else) | Không | Không | Có |
| Block server | Ít hơn | Ít hơn | Có thể (nếu script lớn) |
| Use case | Bulk get/set, batch | Nhóm command không logic | Complex atomic logic |

**Decision tree**:
- Chỉ cần giảm network → **Pipeline**
- Cần nhóm atomic, không logic → **MULTI/EXEC**
- Cần atomic + logic → **Lua**

### 7.5 Pipeline Pitfalls

| Pitfall | Hậu quả | Giải pháp |
|---|---|---|
| Pipeline quá lớn | Memory spike ở client và server | Batch theo nhóm 100-500 |
| Lầm nhầm pipeline với atomic | Race condition | Dùng MULTI/EXEC nếu cần atomic |
| Error handling trong pipeline | Một command lỗi không ngưng cái khác | Kiểm tra kết quả từng command |

```typescript
// Batch pipeline theo chunk để tránh memory spike
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

## 8. Persistence — Giới Thiệu

Day 4 sẽ học sau. Day 2 hiểu concept để biết lúc configure.

### 8.1 RDB (Redis Database Snapshot)

RDB = Periodic snapshot của toàn bộ data, lưu thành binary file `.rdb`.

```
Redis RAM → [BGSAVE mỗi 15 phút] → /data/dump.rdb (binary)
```

**Đặc điểm**:
- File compact, load nhanh khi restart
- Có thể mất data giữa 2 lần snapshot
- Ít ảnh hưởng performance (BGSAVE dùng fork)

**Cấu hình mặc định**:
```
save 3600 1     # Nếu >= 1 key thay đổi trong 1 giờ → snapshot
save 300 100    # Nếu >= 100 key thay đổi trong 5 phút → snapshot
save 60 10000   # Nếu >= 10000 key thay đổi trong 1 phút → snapshot
```

### 8.2 AOF (Append-Only File)

AOF = Ghi mỗi write command vào file log.

```
Redis RAM → [ghi mỗi command] → /data/appendonly.aof (text/binary)
```

**Đặc điểm**:
- Ít mất data hơn RDB (có thể chỉ mất 1 second)
- File lớn hơn RDB
- Có thể rewrite/compact để thu nhỏ (BGREWRITEAOF)

**Fsync options**:
```
appendfsync always      # Flush mỗi command → an toàn nhất, chậm nhất
appendfsync everysec    # Flush mỗi giây → balance tốt (mất tối đa 1 giây data)
appendfsync no          # OS quyết định → nhanh nhất, mất nhiều nhất
```

### 8.3 No Persistence — Pure Cache

```
# redis.conf
save ""           # Tắt RDB
appendonly no     # Tắt AOF
```

Restart → mất tất cả. Dùng cho pure cache layer, data lấy lại từ PostgreSQL được.

### 8.4 Nên Chọn Gì?

| Use case | Persistence mode |
|---|---|
| Pure cache (data lấy lại được) | No persistence |
| Cache + cần recovery nhanh | RDB |
| Session, lock, counter (mất ít OK) | AOF everysec |
| Không muốn mất data | AOF always hoặc RDB + AOF |

**Timekeeping system**:
- Session, OTP, lock → `AOF everysec` (mất tối đa 1 giây OK)
- Employee cache → No persistence (lấy lại từ PostgreSQL khi restart)

---

## 9. Bài Tập Day 2

### Bài 1: Bitmap Attendance

```bash
# Setup: Đánh dấu attendance cho ngày 2026-07-08
SETBIT attendance:2026-07-08 50 1
SETBIT attendance:2026-07-08 100 1
SETBIT attendance:2026-07-08 101 1
SETBIT attendance:2026-07-08 102 1
SETBIT attendance:2026-07-08 200 1

# Câu hỏi:
# 1. Employee 100 đã check-in chưa?
# 2. Tổng có bao nhiêu người check-in?
# 3. Employee nào có ID nhỏ nhất mà chưa check-in?
# 4. Employee nào đã check-in CẢ HAI ngày 07 và 08?

SETBIT attendance:2026-07-07 100 1
SETBIT attendance:2026-07-07 200 1
SETBIT attendance:2026-07-07 300 1
BITOP AND result:both attendance:2026-07-07 attendance:2026-07-08
BITCOUNT result:both
```

### Bài 2: HyperLogLog Dashboard

```bash
# Simulate employees đang mở dashboard
PFADD hll:dashboard:2026-07-08 emp100 emp101 emp102
PFADD hll:dashboard:2026-07-08 emp100   # Trùng lặp
PFADD hll:dashboard:2026-07-08 emp103 emp104

PFCOUNT hll:dashboard:2026-07-08        # Bao nhiêu unique?

# Merge với ngày hôm trước
PFADD hll:dashboard:2026-07-07 emp100 emp200 emp300
PFMERGE hll:dashboard:week hll:dashboard:2026-07-07 hll:dashboard:2026-07-08
PFCOUNT hll:dashboard:week
```

### Bài 3: Pub/Sub Check-in Event

```bash
# Terminal 1: Subscribe
SUBSCRIBE attendance-events

# Terminal 2: Publish check-in events
PUBLISH attendance-events '{"emp_id":100,"type":"check_in","time":"08:00"}'
PUBLISH attendance-events '{"emp_id":101,"type":"check_in","time":"08:05"}'

# Quan sát Terminal 1 nhận message không?

# Thử: Đóng Terminal 1, publish thêm message ở Terminal 2
# Quan sát: message bị mất
```

### Bài 4: Streams Consumer Group

```bash
# Tạo stream và consumer group
XGROUP CREATE stream:attendance-events workers $ MKSTREAM

# Add message
XADD stream:attendance-events * emp_id 100 type check_in
XADD stream:attendance-events * emp_id 101 type check_in
XADD stream:attendance-events * emp_id 102 type check_out

# Worker 1 đọc và xử lý
XREADGROUP GROUP workers worker-1 COUNT 5 STREAMS stream:attendance-events >

# Xem pending (chưa ack)
XPENDING stream:attendance-events workers - + 10

# Ack sau khi xử lý
XACK stream:attendance-events workers {message-id-from-above}

# Xem pending còn lại
XPENDING stream:attendance-events workers - + 10
```

### Bài 5: Transaction MULTI/EXEC

```bash
# Atomic: tăng counter và set TTL
MULTI
INCR stats:checkin:2026-07-08
EXPIRE stats:checkin:2026-07-08 86400
EXEC

# Quan sát: cả 2 command chạy hay chỉ 1?
# Thử lỗi: MULTI, LPUSH stats:checkin:2026-07-08 "foo", INCR stats:checkin:2026-07-08, EXEC
# Quan sát: Redis xử lý lỗi thế nào?
```

### Bài 6: Lua Script Release Lock

```bash
# Setup lock
SET lock:checkin:100:2026-07-08 "my-uuid-abc" NX EX 30

# Release lock đúng cách với Lua
EVAL "if redis.call('GET', KEYS[1]) == ARGV[1] then return redis.call('DEL', KEYS[1]) else return 0 end" 1 lock:checkin:100:2026-07-08 "my-uuid-abc"
# Kết quả: 1 (thành công)

# Thử với sai token
SET lock:checkin:101:2026-07-08 "correct-uuid" NX EX 30
EVAL "if redis.call('GET', KEYS[1]) == ARGV[1] then return redis.call('DEL', KEYS[1]) else return 0 end" 1 lock:checkin:101:2026-07-08 "wrong-uuid"
# Kết quả: 0 (không del, vì token sai)
```

### Bài 7: Pipeline Bulk Cache

```bash
# Dùng redis-cli pipeline mode
# Tạo file commands.txt:
SET cache:employee:1 '{"id":1,"name":"Emp 1"}' EX 600
SET cache:employee:2 '{"id":2,"name":"Emp 2"}' EX 600
SET cache:employee:3 '{"id":3,"name":"Emp 3"}' EX 600
GET cache:employee:1
GET cache:employee:2
GET cache:employee:3

# Chạy pipeline
cat commands.txt | docker exec -i redis-lab redis-cli --pipe

# Hoặc trong redis-cli, dùng MULTI/EXEC để minh họa
MULTI
GET cache:employee:1
GET cache:employee:2
GET cache:employee:3
EXEC
```

---

## 10. Checklist Day 2

### Kiến thức

- [ ] Giải thích Bitmap dùng cho boolean flag, và khi nào chọn Bitmap vs Set
- [ ] Giải thích HyperLogLog là approximate unique count, không lấy được danh sách
- [ ] Giải thích Pub/Sub mất message khi consumer offline — tại sao
- [ ] So sánh Streams vs Pub/Sub: ack, consumer group, replay
- [ ] Giải thích MULTI/EXEC khác SQL transaction thế nào (không có rollback logic)
- [ ] Giải thích WATCH là optimistic locking
- [ ] Giải thích Lua atomic hơn Pipeline
- [ ] Giải thích Pipeline chỉ giảm RTT, không atomic
- [ ] Nêu được 3 persistence mode và khi nào dùng mỗi mode

### Thực hành

- [ ] SETBIT/GETBIT/BITCOUNT cho attendance flag
- [ ] BITOP AND cho intersection 2 ngày
- [ ] PFADD/PFCOUNT/PFMERGE cho HyperLogLog
- [ ] Subscribe và publish trong 2 terminal riêng
- [ ] XADD/XGROUP/XREADGROUP/XACK
- [ ] MULTI/EXEC với check-in counter
- [ ] WATCH pattern
- [ ] Lua release lock
- [ ] Pipeline bulk get/set

---

## 11. Production Pitfalls Day 2

### P1: Pub/Sub Consumer Gap → Mất Message

```
T1: Subscriber đang chạy, nhận message OK
T2: Subscriber restart (deploy mới, crash)
T3: Publisher gửi 50 messages
T4: Subscriber chạy lại — 50 messages đã mất
```

**Giải pháp**: Dùng Streams với consumer group. XREADGROUP + XACK đảm bảo delivery.

---

### P2: MULTI/EXEC Không Có Rollback

```bash
MULTI
INCR budget:department:10    # Giảm budget
INCR purchase:order:500      # Tăng đơn hàng
EXEC
```

Nếu command thứ 2 lỗi runtime (sai type), command thứ 1 đã chạy. Không có rollback.

**Giải pháp**: Với logic phức tạp cần rollback → dùng Lua (có thể trả về error, gọi pcode xử lý), hoặc đơn giản hóa business logic.

---

### P3: Lua Script Dài Block Event Loop

```lua
-- Lua script với loop lớn → NGUY HIỂM
for i=1,100000 do
  redis.call('INCR', 'counter:'..i)
end
```

Redis block trong thời gian này. Tất cả client khác phải đợi.

**Giải pháp**: Giữ Lua script ngắn (< 1ms). Nếu cần xử lý nhiều → chia nhỏ, hoặc xử lý ở application layer.

---

### P4: Pipeline Quá Lớn

```typescript
// BAD: Pipeline 100,000 commands
const pipeline = redis.pipeline();
for (let i = 0; i < 100000; i++) {
  pipeline.set(`key:${i}`, 'value');
}
await pipeline.exec();   // Memory spike ở cả client và server
```

**Giải pháp**: Chunk pipeline theo batch 200-500 commands.

---

### P5: Streams Pending List Phình To

Nếu consumer không XACK, pending list tăng vô hạn.

```bash
# Kiểm tra pending entries
XPENDING stream:attendance-events workers - + 100
XLEN stream:attendance-events

# Monitor XPENDING định kỳ
```

**Giải pháp**: Monitor pending list trong alert system. Implement XCLAIM/XAUTOCLAIM cho message bị stale.

---

### P6: Bitmap Dense vs Sparse

```bash
# BAD: ID gap → tốn memory
SETBIT attendance:2026-07-08 1000000 1    # Allocate 125KB cho 1 bit
SETBIT attendance:2026-07-08 5000000 1    # Allocate 625KB

# GOOD: Map ID thành sequential index nếu ID không dense
# employee_offset = employee_id % max_employees hoặc dùng map
```

---

## 12. Senior Review Questions Day 2

**Q1**: Khi nào chọn Bitmap và khi nào chọn Set để track daily attendance?

> **Answer**: Bitmap tốt khi employee ID là integer, số lượng lớn (>10k), cần tiết kiệm memory, và cần BITOP (AND/OR) để tính intersection/union giữa các ngày. Set tốt khi ID là string, số lượng nhỏ, hoặc cần lấy danh sách ai đã check-in (SMEMBERS). Caution: Bitmap với ID lớn hoặc sparse làm mất lợi thế memory.

**Q2**: Bạn cần đếm unique employees đã mở dashboard trong tháng. Dùng HLL hay Set?

> **Answer**: HLL nếu scale lớn (>100k unique) và chỉ cần count, không cần biết danh sách ai. Set nếu cần exact count, cần biết cụ thể ai, hoặc scale nhỏ (< 10k unique). Với timekeeping 2000 employee, Set là đủ — không đáng cầu phức tạp HLL.

**Q3**: Tại sao Pub/Sub không phù hợp cho attendance event processing?

> **Answer**: Consumer offline → mất message (không buffer). Không có ack → không biết consumer xử lý thành công chưa. Không có retry khi consumer fail. Không có consumer group để scale. Dùng Streams với XREADGROUP + XACK. Pub/Sub chỉ dùng cho notification nhẹ (realtime dashboard update) có thể mất được.

**Q4**: Mô tả WATCH và ứng dụng cụ thể trong timekeeping.

> **Answer**: WATCH implement optimistic locking — nếu key bị thay đổi sau WATCH và trước EXEC, EXEC trả về nil (abort). Ứng dụng: decrement OTP quota của employee. WATCH quota:emp:100, GET current value, MULTI + DECR + EXEC. Nếu employee dùng 2 device cùng lúc, một trong hai request sẽ fail và phải retry. An toàn hơn Lua cho use case đơn giản.

**Q5**: Giải thích Pipeline vs MULTI/EXEC vs Lua cho batch cache set.

> **Answer**: Pipeline: Gửi tất cả SET cùng lúc, 1 RTT thay vì N RTT. Không atomic — OK cho batch cache set vì từng cache entry là độc lập. MULTI/EXEC: Atomic nhóm SET, nhưng overhead của MULTI/EXEC không cần thiết cho cache set độc lập. Lua: Atomic với logic phức tạp. Over-engineering cho simple batch set. Kết luận: Dùng Pipeline cho batch cache operations thông thường.

**Q6**: Streams consumer group — điều gì xảy ra khi worker-1 crash sau khi XREADGROUP nhưng trước XACK?

> **Answer**: Message nằm trong pending list, XPENDING cho thấy worker-1 đang giữ message đó. Nếu có heartbeat/monitor, worker-2 có thể XCLAIM message sau timeout (vd: 30s). Hoặc XAUTOCLAIM tự động claim. Nếu không có, message bị stuck → implement monitor và auto-claim là mandatory trong production.

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
PFADD key element [element ...]    # Return 1 nếu HLL changed, 0 nếu không
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

*Day 2 hoàn thành. Next: Day 3 — Redis Cluster, Sentinel & High Availability.*
