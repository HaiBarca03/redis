# Day 4: Production Redis — Operations Guide

> **Mindset**: Day 4 là sự khác biệt giữa "Redis chạy được" và "Redis chạy đúng trong production". Một Redis cấu hình sai có thể mất data, bị exploit, hoặc down hết cluster lúc cao điểm.

---

## Mục Tiêu

Sau Day 4, bạn phải:

- [ ] Giải thích RDB vs AOF và chọn đúng cho từng use case
- [ ] Config maxmemory và eviction policy phù hợp
- [ ] Mô tả replication lag và hậu quả của nó
- [ ] Giải thích Sentinel vs Cluster — khi nào dùng cái nào
- [ ] Đọc và hiểu output của `INFO`, `SLOWLOG`, `LATENCY DOCTOR`
- [ ] Tính được hit ratio và biết ngưỡng cảnh báo
- [ ] Setup ACL đúng với least privilege
- [ ] Thiết kế production Redis cho timekeeping system

**Context Project**: Timekeeping System. 500-2000 employees. Peak load 8:00-8:30 AM. Critical data trong PostgreSQL, Redis là cache/coordination layer.

---

## Keywords Day 4

```
Redis persistence RDB AOF appendfsync
Redis BGSAVE BGREWRITEAOF RDB file dump.rdb
Redis appendonly.aof aof-rewrite-min-size
Redis maxmemory eviction policy
Redis allkeys-lru allkeys-lfu volatile-lru volatile-lfu volatile-ttl noeviction
Redis replication replica lag replication offset
Redis Sentinel failover quorum
Redis Cluster hash slot hash tag MOVED ASK
Redis INFO sections: memory server keyspace replication
Redis SLOWLOG GET latency-monitor-threshold
Redis CLIENT LIST CLIENT KILL
Redis LATENCY DOCTOR LATENCY HISTORY
Redis ACL AUTH ACL WHOAMI
Redis TLS protected-mode requirepass
Redis production checklist
Redis managed cloud: ElastiCache Upstash Redis Cloud
```

---

## 1. Persistence

### 1.1 RDB (Redis Database Snapshot)

RDB = Periodic fork và dump toàn bộ data ra file nhị phân.

```
Redis RAM → fork() → child process → serialize → /data/dump.rdb
```

**Cấu hình:**

```conf
# redis.conf
# Snapshot nếu >= 1 key thay đổi trong 3600 giây (1 giờ)
save 3600 1
# Snapshot nếu >= 100 key thay đổi trong 300 giây (5 phút)
save 300 100
# Snapshot nếu >= 10000 key thay đổi trong 60 giây (1 phút)
save 60 10000

# Tắt RDB
save ""

# File path
dir /data
dbfilename dump.rdb

# Nếu RDB fail → từ chối write (an toàn hơn)
stop-writes-on-bgsave-error yes

# Compress RDB file (CPU vs disk trade-off)
rdbcompression yes
```

**Manual trigger:**

```bash
BGSAVE          # Background snapshot (non-blocking)
SAVE            # Foreground snapshot (blocking — không dùng production)
LASTSAVE        # Unix timestamp của lần snapshot cuối
```

**Đặc điểm:**

| Tiêu chí | RDB |
|---|---|
| Data mất tối đa | Giữa 2 lần snapshot (có thể 1-60 phút) |
| Tốc độ restart | Nhanh (load binary file) |
| Kích thước file | Nhỏ, compact |
| CPU ảnh hưởng | fork() tạo spike nhỏ (có thể vài trăm ms) |
| Use case | Backup, disaster recovery |

---

### 1.2 AOF (Append-Only File)

AOF = Ghi mỗi write command vào file log. Khi restart, replay lại toàn bộ log.

```
SET key val   →  [*3\r\n$3\r\nSET\r\n$3\r\nkey\r\n$3\r\nval\r\n] → appendonly.aof
INCR counter  →  [ghi vào cuối file]
DEL key       →  [ghi vào cuối file]
```

**Cấu hình:**

```conf
# redis.conf
appendonly yes
appendfilename "appendonly.aof"
dir /data

# Fsync strategy — QUAN TRỌNG NHẤT
appendfsync always      # Flush mỗi command, fsync ngay → an toàn nhất, CHẬM nhất (~1000 ops/s)
appendfsync everysec    # Flush + fsync mỗi giây → BALANCE tốt (mất tối đa 1 giây data)
appendfsync no          # Để OS quyết định (thường 30s) → NHANH nhất, mất nhiều nhất

# AOF rewrite (compact lại file lớn)
auto-aof-rewrite-percentage 100    # Rewrite khi AOF lớn gấp đôi baseline
auto-aof-rewrite-min-size 64mb     # Chỉ rewrite nếu file > 64MB

# An toàn khi AOF bị truncated (crash giữa khi ghi)
aof-use-rdb-preamble yes           # Hybrid: RDB snapshot + AOF delta (Redis 4.0+)
```

**Manual rewrite:**

```bash
BGREWRITEAOF    # Compact AOF file (background, non-blocking)
```

**Đặc điểm:**

| Tiêu chí | AOF everysec | AOF always | AOF no |
|---|---|---|---|
| Data mất tối đa | 1 giây | 0 (gần như) | Nhiều giây |
| Tốc độ ghi | Cao | Thấp (~1k ops/s) | Cao nhất |
| Kích thước file | Lớn | Lớn | Lớn |
| Restart speed | Chậm (replay log) | Chậm | Chậm |

---

### 1.3 No Persistence

```conf
save ""
appendonly no
```

Restart → mất toàn bộ. Dùng cho **pure cache** — data lấy lại từ PostgreSQL khi miss.

---

### 1.4 Nên Chọn Gì? (Decision Matrix)

| Use Case | Persistence | Lý Do |
|---|---|---|
| Employee profile cache | **None** | DB là source of truth, cache miss OK |
| Attendance summary cache | **None** | Rebuild được từ DB |
| Session tokens | **AOF everysec** | Mất session → user bị kick → bad UX |
| OTP codes | **AOF everysec** | Mất OTP → user không login được |
| Distributed locks | **None / AOF everysec** | Lock ngắn (10s), mất lock = TTL expire tự giải quyết |
| Rate limit counters | **None** | Mất counter = window reset = chấp nhận được |
| Idempotency keys | **AOF everysec** | Mất key → duplicate write → nguy hiểm |
| Stream events | **AOF everysec** | Mất event = mất dữ liệu nghiệp vụ |

**Timekeeping recommendation:**

```conf
# Single Redis instance cho mọi thứ
appendonly yes
appendfsync everysec
save ""          # Tắt RDB, dùng AOF thuần
aof-use-rdb-preamble yes  # Hybrid mode
```

**Hoặc** dùng 2 Redis instance:
- `redis-cache`: No persistence, allkeys-lru (pure cache)
- `redis-state`: AOF everysec, noeviction (session/lock/stream)

---

### 1.5 RDB + AOF Hybrid (Redis 4.0+)

```conf
aof-use-rdb-preamble yes
```

File AOF bắt đầu bằng RDB snapshot → append AOF commands từ đó trở đi. Restart nhanh hơn AOF thuần, dữ liệu đầy đủ hơn RDB thuần.

---

## 2. Memory & Eviction

### 2.1 maxmemory

```conf
# Đặt giới hạn memory
maxmemory 2gb
maxmemory 512mb
maxmemory 0       # Không giới hạn (nguy hiểm trên server có limit)
```

**Ước tính memory cần thiết:**

```
Timekeeping system:
- 2000 employees × 1KB per profile cache = 2MB
- 2000 sessions × 500B per session = 1MB
- Streams: 50000 messages × 200B per message = 10MB
- Leaderboard, counters, locks: ~5MB
- Redis overhead: ~30MB

Tổng: ~50MB với normal load
→ Đặt maxmemory = 256mb (buffer nhiều)
→ Alert khi > 70% (180MB)
→ Check khi > 85% (218MB)
```

```bash
# Kiểm tra hiện tại
INFO memory

# Output quan trọng:
# used_memory:           50331648   (50MB đang dùng)
# used_memory_peak:      78643200   (78MB peak)
# maxmemory:             268435456  (256MB giới hạn)
# mem_fragmentation_ratio: 1.2      (OK, < 1.5 là bình thường)
```

**mem_fragmentation_ratio:**
- `1.0 - 1.5`: Bình thường
- `> 1.5`: Memory fragmentation — Redis đang dùng nhiều hơn cần
- `< 1.0`: Redis đang swap ra disk — NGUY HIỂM, latency tăng vọt

### 2.2 Eviction Policies

Khi Redis đạt `maxmemory`, nó phải quyết định xóa key nào:

```conf
maxmemory-policy allkeys-lru
```

| Policy | Mô Tả | Dùng Khi |
|---|---|---|
| `noeviction` | Từ chối write, trả lời OOM error | Không muốn mất data, sẽ alert và xử lý thủ công |
| `allkeys-lru` | Xóa key ít sử dụng nhất (LRU) trong tất cả key | Pure cache, mọi key có thể bị xóa |
| `allkeys-lfu` | Xóa key ít truy cập nhất (LFU) trong tất cả key | Cache với access pattern không đều |
| `allkeys-random` | Xóa key ngẫu nhiên | Không nên dùng |
| `volatile-lru` | LRU trong các key có TTL | Mixed: có key cần giữ, có key có thể xóa |
| `volatile-lfu` | LFU trong các key có TTL | Mixed với LFU |
| `volatile-ttl` | Xóa key có TTL ngắn nhất trước | Muốn xóa key sắp hết hạn |
| `volatile-random` | Random trong key có TTL | Hiếm khi dùng |

**LRU vs LFU:**
- **LRU** (Least Recently Used): Xóa key **lâu nhất chưa được truy cập**. Phù hợp với temporal access pattern.
- **LFU** (Least Frequently Used): Xóa key **ít được truy cập nhất**. Phù hợp với hot/cold data distribution.

**Timekeeping recommendation:**

```conf
# Redis cache instance
maxmemory-policy allkeys-lru

# Redis state instance (session, lock, stream)
maxmemory-policy noeviction   # Không tự động xóa, alert khi đầy
```

**Lưu ý**: Với `noeviction`, khi đầy, Redis trả lời `OOM command not allowed` cho mọi write. Phải monitor và xử lý trước khi đầy.

---

## 3. Replication

### 3.1 Primary-Replica

```
       WRITE                     READ (optional)
Client ─────→ Primary ─────────→ Replica 1
                     └─────────→ Replica 2
                     
Replication: ASYNC (default)
```

**Cấu hình Primary:**

```conf
# primary không cần thêm config đặc biệt
# Chỉ cần replica biết địa chỉ primary
bind 0.0.0.0
protected-mode yes
requirepass "strong-password-here"
```

**Cấu hình Replica:**

```conf
replicaof 192.168.1.10 6379
masterauth "strong-password-here"
replica-read-only yes    # Replica chỉ cho phép READ
```

**Kiểm tra replication:**

```bash
# Trên primary
INFO replication
# Output:
# role:master
# connected_slaves:2
# slave0:ip=192.168.1.11,port=6379,state=online,offset=1234567,lag=0
# slave1:ip=192.168.1.12,port=6379,state=online,offset=1234560,lag=1
# master_replid:abc123...
# master_repl_offset:1234567
# repl_backlog_size:1048576
```

### 3.2 Replication Lag

**Replication là ASYNC** — replica có thể lag sau primary.

```
Primary: WRITE → SET key 100         (offset: 1000)
Replica: SYNC  ... (lag 50ms) ...    (offset: 950)

If client reads from replica: có thể thấy giá trị cũ
```

**Hậu quả thực tế:**
- Client đọc từ replica: có thể thấy dữ liệu cũ
- Failover: nếu primary crash, replica chưa sync → mất các write gần nhất

**Kiểm tra lag:**

```bash
# Trên primary
INFO replication | grep lag
# slave0:ip=...,lag=0   → Đồng bộ tốt
# slave0:ip=...,lag=5   → Lag 5 giây → cần kiểm tra

# Thiết lập threshold alert
redis.conf: min-slaves-max-lag 10
# Primary từ chối write nếu tất cả replica lag > 10 giây
```

**Xử lý Replication Lag trong App:**

```typescript
// Strategy 1: Luôn đọc từ primary (đơn giản, tốt cho consistency)
// Dùng primary cho cả READ và WRITE

// Strategy 2: Read-your-own-writes
// Sau write → đọc từ primary (hoặc đợi replica sync)
// Với IOredis:
const primaryClient = new Redis({ host: 'primary', port: 6379 });
const replicaClient = new Redis({ host: 'replica', port: 6379 });

// Chỉ dùng replica cho data không critical
async getAttendanceSummary(empId: number, date: string) {
  // Stale OK → read from replica
  return replicaClient.get(`cache:attendance-summary:${empId}:${date}`);
}

async getSessionData(token: string) {
  // Session phải chính xác → read from primary
  return primaryClient.get(`session:${token}`);
}
```

---

### 3.3 Sentinel — High Availability

Sentinel giải quyết: **Primary down → tự động failover sang Replica**.

```
                    ┌─────────────────────────┐
                    │   Sentinel 1 (leader)   │ ← Elect leader
                    │   Sentinel 2            │ ← Monitor
                    │   Sentinel 3            │ ← Monitor
                    └─────────────────────────┘
                              │ Monitor & Failover
                    ┌─────────┴──────────┐
                Primary         Replica 1         Replica 2
                  ↓ crash              ↑
                           Promote to Primary
```

**Yêu cầu tối thiểu**: 3 Sentinel (để bầu ra quorum majority = 2/3).

**Cấu hình Sentinel:**

```conf
# sentinel.conf
sentinel monitor mymaster 192.168.1.10 6379 2
# mymaster = tên (đặt tùy ý)
# 192.168.1.10 6379 = primary address
# 2 = số sentinel đồng ý để failover (quorum)

sentinel auth-pass mymaster "strong-password"
sentinel down-after-milliseconds mymaster 5000     # 5s không pong → subjectively down
sentinel failover-timeout mymaster 60000           # Timeout failover sau 60s
sentinel parallel-syncs mymaster 1                 # Mỗi lúc 1 replica sync (để giảm tải)
```

**App kết nối Sentinel:**

```typescript
// ioredis với Sentinel
import Redis from 'ioredis';

const redis = new Redis({
  sentinels: [
    { host: '192.168.1.20', port: 26379 },
    { host: '192.168.1.21', port: 26379 },
    { host: '192.168.1.22', port: 26379 },
  ],
  name: 'mymaster',        // Tên khai báo trong sentinel.conf
  password: 'strong-password',
  sentinelPassword: 'sentinel-password',
  retryStrategy: (times) => Math.min(times * 50, 2000), // Retry với backoff
});

redis.on('error', (err) => console.error('Redis error:', err));
redis.on('reconnecting', () => console.log('Redis reconnecting...'));
```

**Sentinel giới hạn:**
- Sentinel là **HA, không phải sharding**
- Vẫn chỉ 1 primary cho write → vertical scale only
- Failover mất vài giây → app phải xử lý retry

---

## 4. Redis Cluster

### 4.1 Hash Slots

Redis Cluster phân tán data qua **16384 hash slots**.

```
HASH_SLOT = CRC16(key) % 16384

Cluster 3 primary nodes:
Node A: slots 0     - 5460     (node 192.168.1.10)
Node B: slots 5461  - 10922    (node 192.168.1.11)
Node C: slots 10923 - 16383    (node 192.168.1.12)

Key "employee:100" → CRC16("employee:100") % 16384 = 7638 → Node B
Key "session:abc"  → CRC16("session:abc") % 16384  = 1234 → Node A
```

**MOVED redirect**: Nếu client gọi sai node, Redis trả lời `MOVED`:

```
Client → Node A: GET employee:100
Node A: -MOVED 7638 192.168.1.11:6379
Client → Node B: GET employee:100 (redirect)
```

Smart Redis clients (ioredis) tự xử lý MOVED redirect.

### 4.2 Hash Tag

Nếu cần 2 key cùng nằm 1 slot (để dùng multi-key command), dùng **hash tag**: phần trong `{}` được dùng để tính hash.

```bash
# Không có hash tag: khác slot → không thể dùng MGET/MULTI trong cluster
employee:100:profile   → CRC16("employee:100:profile") → slot A
employee:100:session   → CRC16("employee:100:session") → slot B

# Có hash tag: cùng slot → có thể dùng MGET/MULTI
user:{100}:profile     → CRC16("100") → slot X
user:{100}:session     → CRC16("100") → slot X (cùng slot!)

# Thực hành với timekeeping
cache:{emp:100}:profile
cache:{emp:100}:schedule
cache:{emp:100}:attendance
```

```bash
# Với hash tag cùng slot → MGET hợp lệ trong cluster
MGET user:{100}:profile user:{100}:session  # OK
MGET employee:100 session:100               # ERROR (khác slot)
```

### 4.3 Multi-key Limitation trong Cluster

```bash
# CÁC LỆNH KHÔNG HOẠT ĐỘNG CROSS-SLOT
MGET key1 key2          # Chỉ OK nếu cùng slot
MSET key1 val key2 val  # Chỉ OK nếu cùng slot
SUNIONSTORE dest k1 k2  # Chỉ OK nếu cùng slot
MULTI
  INCR key1             # Cross-slot in MULTI → ERROR
  SET key2 val
EXEC
```

**Giải pháp:**

```typescript
// Thay vì MGET cross-slot, dùng pipeline individual GETs
const pipeline = redis.pipeline();
ids.forEach(id => pipeline.get(`cache:employee:${id}`));
const results = await pipeline.exec();
// Pipeline hoạt động tốt trong cluster (mỗi command routing riêng)
```

### 4.4 Cluster vs Sentinel vs Standalone

| Tiêu chí | Standalone | Sentinel | Cluster |
|---|---|---|---|
| HA (High Availability) | Không | Có | Có |
| Horizontal sharding | Không | Không | Có |
| Multi-key commands | Đầy đủ | Đầy đủ | Giới hạn (same slot) |
| Complexity | Thấp | Trung bình | Cao |
| Scale-out | Không | Không | Có |
| Khi dùng | Dev, small prod | Prod, single shard | Prod, nhiều GB data |

**Timekeeping recommendation**: **Sentinel** trên 2-3 server. Data không lớn (< 1GB), Cluster thêm phức tạp không cần thiết. Hoặc dùng **managed Redis** (ElastiCache, Upstash) để tránh operational burden.

---

## 5. Monitoring

### 5.1 INFO Sections

```bash
INFO                    # Tất cả sections
INFO server             # Version, OS, port, uptime
INFO clients            # Số kết nối, blocked clients
INFO memory             # Memory usage, fragmentation
INFO persistence        # RDB/AOF status
INFO stats              # Command count, hit/miss, eviction
INFO replication        # Primary/replica status, lag
INFO keyspace           # Số key per DB, TTL info
```

**Các metric quan trọng cần theo dõi:**

```bash
INFO memory
# used_memory:                50331648     # Đang dùng 50MB
# used_memory_rss:            67108864     # OS thấy 64MB (RSS > used = fragmentation)
# mem_fragmentation_ratio:    1.33         # OK (< 1.5)
# mem_fragmentation_bytes:    12345678     # Số byte bị fragment

INFO stats
# instantaneous_ops_per_sec:  12345        # Hiện tại 12k ops/s
# total_commands_processed:   987654321    # Tổng số command từ khi khởi động
# keyspace_hits:              900000       # 900k cache hits
# keyspace_misses:            100000       # 100k cache misses
# expired_keys:               5000         # Số key đã expire
# evicted_keys:               0            # Số key bị evict (CẢNH BÁO nếu > 0 liên tục)
# rejected_connections:       0            # Số kết nối bị từ chối (CẢNH BÁO nếu > 0)

INFO clients
# connected_clients:          150          # Hiện tại 150 connection
# blocked_clients:            0            # Không có client bị block
# maxclients:                 10000        # Giới hạn kết nối

INFO replication
# role:master
# connected_slaves:           2
# slave0:ip=...,lag:          0            # CẢNH BÁO nếu lag > 5s
```

### 5.2 Hit Ratio

```
Hit Ratio = keyspace_hits / (keyspace_hits + keyspace_misses)
```

```bash
# Lấy từ INFO stats
redis-cli info stats | grep -E "keyspace_hits|keyspace_misses"

# Tính toán
hits = 900000
misses = 100000
hit_ratio = 900000 / (900000 + 100000) = 0.90 = 90%
```

| Hit Ratio | Đánh Giá | Hành Động |
|---|---|---|
| > 90% | Tốt | Bình thường |
| 80-90% | Trung bình | Xem lại TTL, cache strategy |
| 70-80% | Thấp | Review key naming, data type |
| < 70% | Kém | Cache ko hiệu quả, review architecture |

**Timekeeping target**: > 85% vào bình thường, > 75% vào peak 8h sáng (nhiều cache miss do check-in mới).

### 5.3 SLOWLOG

```bash
# Cấu hình
redis.conf:
slowlog-log-slower-than 10000   # Log command chậm hơn 10ms (10000 microseconds)
slowlog-max-len 128              # Giữ tối đa 128 entry

# Xem slow log
SLOWLOG GET 10          # 10 entries gần nhất
SLOWLOG GET             # Tất cả
SLOWLOG LEN             # Số entry hiện tại
SLOWLOG RESET           # Xóa slowlog

# Format output:
# 1) (integer) 14              ← ID
# 2) (integer) 1720401600      ← Unix timestamp
# 3) (integer) 15234           ← Thời gian thực thi (microseconds)
# 4) 1) "KEYS"                 ← Command
#    2) "*"                    ← Arguments ← ĐÂY CHÍNH LÀ VẤN ĐỀ
# 5) "127.0.0.1:54321"        ← Client
# 6) ""                        ← Client name
```

**Phân tích SLOWLOG:**
- `KEYS *` → Thay bằng `SCAN`
- `SMEMBERS large-set` → Dùng `SSCAN`
- `HGETALL large-hash` → Giữ hash nhỏ, dùng `HMGET` field cụ thể
- `ZRANGE 0 -1` trên large ZSet → Thêm `COUNT` limit

### 5.4 LATENCY

```bash
# Bật latency monitoring
redis.conf:
latency-monitor-threshold 100    # Log event trễ hơn 100ms

# Commands
LATENCY LATEST              # Các event trễ gần nhất
LATENCY HISTORY event-name  # Lịch sử của event cụ thể
LATENCY RESET               # Reset tất cả
LATENCY DOCTOR              # Phân tích và gợi ý tự động (rất hữu ích)

# LATENCY DOCTOR output example:
# I detected 2 latency samples:
# 1. fork: 354ms. ADVICE: Consider lowering rdb-save-incremental-fsync to yes
# 2. aof_write_pending_fsync: 12ms. ADVICE: Use appendfsync everysec instead of always
```

### 5.5 CLIENT LIST

```bash
CLIENT LIST                     # Tất cả client đang kết nối
CLIENT LIST TYPE normal         # Loại: normal, pubsub, slave, monitor
CLIENT KILL ID 12345            # Ngắt kết nối client cụ thể
CLIENT SETNAME myapp-worker-1   # Đặt tên cho connection (debug dễ hơn)
CLIENT GETNAME                  # Lấy tên hiện tại

# Client LIST output:
# id=5 addr=127.0.0.1:54321 fd=12 name=myapp-worker cmd=brpop age=120 idle=0 ...
# id=6 addr=127.0.0.1:54322 fd=13 name= cmd=subscribe age=3600 idle=3600 ...
```

### 5.6 Dashboard Monitoring

**Metrics cần alert:**

| Metric | Ngưỡng Cảnh Báo | Ngưỡng Critical | Hành Động |
|---|---|---|---|
| `used_memory` | > 70% maxmemory | > 85% maxmemory | Tăng maxmemory hoặc xóa key |
| `mem_fragmentation_ratio` | > 1.5 | > 2.0 | MEMORY PURGE hoặc restart |
| `evicted_keys` | > 0 (liên tục) | > 100/phút | Tăng maxmemory hoặc review TTL |
| `rejected_connections` | > 0 | > 0 | Tăng maxclients, check connection pool |
| `blocked_clients` | > 10 | > 50 | Xem BLPOP/BRPOP timeout |
| `replication lag` | > 2s | > 10s | Kiểm tra mạng, xem BGSAVE |
| `slowlog entries` | > 10/phút | > 50/phút | Xem SLOWLOG, tìm command lỗi |
| `hit_ratio` | < 80% | < 70% | Review cache strategy |
| `instantaneous_ops_per_sec` | > 50k | > 100k | Review workload, scale |

**Prometheus + Grafana** (Recommended):

```yaml
# docker-compose.yml
services:
  redis-exporter:
    image: oliver006/redis_exporter
    environment:
      REDIS_ADDR: redis://redis:6379
      REDIS_PASSWORD: strong-password
    ports:
      - "9121:9121"
```

Các dashboard có sẵn: `grafana.com/dashboards/11835` (Redis Sentinel) hoặc `grafana.com/dashboards/763`.

---

## 6. Security

### 6.1 Network Security — Quan Trọng Nhất

```conf
# TUYỆT ĐỐI KHÔNG bind 0.0.0.0 nếu expose ra internet
bind 127.0.0.1 192.168.1.10    # Chỉ nghe trên interface nội bộ
protected-mode yes              # Tự động bật nếu không có bind và password
port 6379

# Đổi port mặc định (security through obscurity, ít hiệu quả nhưng thêm 1 lớp)
port 6380

# Disable các command nguy hiểm
rename-command FLUSHALL ""      # Disable FLUSHALL
rename-command FLUSHDB ""       # Disable FLUSHDB
rename-command DEBUG ""         # Disable DEBUG
rename-command CONFIG "CONFIG_SECURE_KEY_abc123"  # Đổi tên CONFIG
```

### 6.2 Authentication

```conf
# Legacy: requirepass (tất cả user cùng password)
requirepass "VeryStr0ng-P@ssw0rd-2026!"

# Modern: ACL (Redis 6+)
aclfile /etc/redis/acl.conf

# Tắt default user
ACL SETUSER default off
```

### 6.3 ACL (Access Control List)

ACL cho phép tạo user với quyền cụ thể.

```bash
# Xem user hiện tại
ACL LIST
ACL WHOAMI
ACL CAT         # List tất cả command categories
ACL CAT string  # List tất cả command trong category string

# Tạo user cho app (chỉ được dùng key cache:*)
ACL SETUSER app-user on >AppStr0ngP@ss ~cache:* +get +set +del +expire +exists +ttl +type
# on         = user active
# >password  = password
# ~cache:*   = chỉ được truy cập key bắt đầu bằng cache:
# +get +set  = chỉ được dùng các command này

# User đọc toàn bộ (chỉ GET, không write)
ACL SETUSER readonly-user on >ReadOnlyP@ss ~* +get +mget +hget +hmget +hgetall +lrange +smembers +zrange

# User full quyền cho ops team (với giới hạn key không quá rộng)
ACL SETUSER ops-admin on >OpsP@ss ~* +@all

# Save ACL ra file
ACL SAVE

# Load lại ACL từ file
ACL LOAD
```

**ACL cho Timekeeping System:**

```bash
# App service user: full CRUD trên namespace của nó
ACL SETUSER timekeeping-app on >AppP@ss \
  ~cache:* ~session:* ~otp:* ~rate:* ~lock:* ~stats:* ~online:* ~leaderboard:* \
  ~idempotency:* ~stream:attendance-events \
  +get +set +del +expire +exists +ttl \
  +hget +hset +hdel +hincrby +hgetall \
  +zadd +zrem +zrange +zrank +zscore +zcard +zremrangebyscore \
  +sadd +srem +sismember +scard \
  +incr +incrby +decr \
  +xadd +xread +xgroup +xreadgroup +xack +xpending +xautoclaim \
  +eval +evalsha \
  -@dangerous

# Worker service user: chỉ stream
ACL SETUSER stream-worker on >WorkerP@ss \
  ~stream:* \
  +xreadgroup +xack +xpending +xautoclaim +xclaim +xadd +xdel

# Monitoring user: chỉ read-only
ACL SETUSER monitoring on >MonitorP@ss ~* \
  +info +dbsize +slowlog +client|list +memory|usage \
  +get +hgetall +type +ttl +exists
```

### 6.4 TLS

```conf
# redis.conf
tls-port 6380
port 0              # Disable non-TLS
tls-cert-file /etc/ssl/redis/redis.crt
tls-key-file /etc/ssl/redis/redis.key
tls-ca-cert-file /etc/ssl/redis/ca.crt
tls-auth-clients yes   # Yêu cầu client certificate
```

```typescript
// ioredis với TLS
const redis = new Redis({
  host: 'redis.internal',
  port: 6380,
  tls: {
    ca: fs.readFileSync('/etc/ssl/ca.crt'),
    cert: fs.readFileSync('/etc/ssl/client.crt'),
    key: fs.readFileSync('/etc/ssl/client.key'),
  },
  password: 'strong-password',
});
```

### 6.5 Security Checklist

- [ ] Không expose Redis ra public internet
- [ ] Bind trên private interface only
- [ ] `protected-mode yes`
- [ ] Dùng ACL, disable default user
- [ ] Password mạnh (>20 chars)
- [ ] TLS trong cloud/multi-datacenter
- [ ] Disable FLUSHALL, FLUSHDB (rename-command)
- [ ] Không log sensitive data (OTP, session token) trong app logs
- [ ] Network firewall: chỉ allow app servers truy cập Redis port
- [ ] Monitor connected_clients bất thường (có thể là intrusion)

---

## 7. Operational Pitfalls

### P1: Fork Lag trong BGSAVE / BGREWRITEAOF

```
Redis dùng fork() để BGSAVE
fork() phải copy page table của process hiện tại
Nếu Redis đang dùng 2GB → fork() mất 1-2 giây → tất cả command bị block
```

**Giải pháp:**
- Cấu hình: `no-appendfsync-on-rewrite yes` — tránh AOF fsync trong khi rewrite
- Đặt BGSAVE vào giờ ít tải
- Dùng smaller Redis instances thay vì 1 instance lớn

---

### P2: Maxmemory Đặt Trên Tổng RAM

```
Server có 4GB RAM
maxmemory = 0 (không giới hạn) → Redis dùng hết → OS swap → latency vọt
```

**Rule**: `maxmemory <= 50-60% total RAM` để dành cho OS và process khác.

---

### P3: Replication Full Resync

```
Replica bị ngắt kết nối > repl-backlog-size / tốc độ write
→ Redis phải FULL RESYNC (dump lại toàn bộ data)
→ Trong thời gian đó: CPU/disk/memory spike ở Primary
```

**Giải pháp:**

```conf
# Tăng repl-backlog
repl-backlog-size 512mb   # Mặc định 1MB → tăng lên nếu replica hay bị ngắt
repl-timeout 60           # Timeout cho replication connection
```

---

### P4: Cluster Resharding Làm Gián Đoạn

Khi thêm node vào cluster, phải resharding (di chuyển hash slot). Trong quá trình đó, một số key không truy cập được.

**Giải pháp**: Resharding vào giờ low traffic. Dùng `redis-cli --cluster reshard` với `--cluster-pipeline` để tăng tốc. Làm từng batch nhỏ.

---

### P5: Không Monitor Slowlog

`KEYS *` bị chạy trong production (code cũ, debug script) → block 500ms → tất cả request treo → timeout → incident.

**Giải pháp**: Đặt alert khi SLOWLOG có entry > 10ms. Review code trước deploy: tìm KEYS, SMEMBERS, HGETALL.

---

### P6: Connection Pool Quá Lớn

```
10 NestJS instances × 100 Redis connections = 1000 connections
Redis mặc định maxclients = 10000 → OK
Nhưng mỗi connection tốn ~10KB memory → 1000 connections = 10MB overhead
```

**Giải pháp**: Ioredis sử dụng single connection per instance (multiplexed). Không tạo nhiều Redis instances trong code. Dùng connection pool đúng cách.

```typescript
// BAD: Tạo mới Redis instance mỗi request
async getEmployee(id: number) {
  const redis = new Redis({ host: '...', port: 6379 }); // Mỗi lần tạo mới!
  const result = await redis.get(`cache:employee:${id}`);
  await redis.quit();
  return result;
}

// GOOD: Dùng singleton
@Module({
  providers: [{
    provide: 'REDIS_CLIENT',
    useFactory: () => new Redis({ host: '...', port: 6379 }),
  }]
})
```

---

### P7: Không Đặt TTL cho Cache Key

Memory tăng theo thời gian → eviction bắt đầu xóa key → cache hit ratio giảm đột ngột → DB bị hit nặng.

```bash
# Kiểm tra key không có TTL
SCAN 0 COUNT 100
# Với mỗi key: TTL key → nếu -1 thì không có TTL

# Kiểm tra nhanh
redis-cli --scan --pattern "cache:*" | xargs -L 1 redis-cli TTL | grep -c "^-1"
# Đếm số cache key không có TTL
```

---

## 8. Bài Tập Day 4: Thiết Kế Production Redis cho Timekeeping

### 8.1 Use Cases & Key Types

| Use Case | Keys | Type | TTL | Persistence Needed? |
|---|---|---|---|---|
| Employee profile cache | `cache:employee:{id}` | String (JSON) | 10 phút | Không |
| Shift config cache | `cache:shift:{id}` | String (JSON) | 60 phút | Không |
| Attendance summary cache | `cache:attendance:{id}:{date}` | String (JSON) | 2 phút | Không |
| Session token | `session:{token}` | String | 8 giờ | Có (AOF) |
| OTP code | `otp:{phone}` | String | 3 phút | Có (AOF) |
| OTP attempt | `otp:attempt:{phone}` | String (counter) | 15 phút | Không |
| Login rate limit | `rate:login:{id}` | String (counter) | 15 phút | Không |
| Check-in rate limit | `rate:checkin:{id}` | String (counter) | 1 phút | Không |
| Distributed lock | `lock:{resource}` | String | 10 giây | Không |
| Idempotency key | `idempotency:{path}:{key}` | String | 24 giờ | Có (AOF) |
| Daily attendance bitmap | `attendance:{date}` | Bitmap | 25 giờ | Không |
| Online employees | `online:employees` | Set | No TTL | Không |
| Check-in leaderboard | `leaderboard:early-checkin:{date}` | ZSet | 25 giờ | Không |
| Daily check-in counter | `stats:checkin:{date}` | String | 25 giờ | Không |
| Attendance stream | `stream:attendance-events` | Stream | MAXLEN ~50k | Có (AOF) |

### 8.2 Kiến Trúc Đề Xuất

```
                    ┌─────────────────────────┐
   NestJS App       │   Redis Sentinel Setup  │
   (3 instances)    │                         │
        │           │   Primary (192.168.1.10)│
        └──────────→│   Replica (192.168.1.11)│
   IOredis client   │   Replica (192.168.1.12)│
   (Sentinel aware) │                         │
                    │   Sentinel × 3          │
                    └─────────────────────────┘
                    
Server spec per node: 2 vCPU, 4GB RAM, SSD
Redis maxmemory: 2GB (50% of RAM)
```

### 8.3 Cấu Hình Đề Xuất

```conf
# redis.conf cho Primary

# Network
bind 192.168.1.10 127.0.0.1
protected-mode yes
port 6379

# Memory
maxmemory 2gb
maxmemory-policy volatile-lru

# Persistence
appendonly yes
appendfsync everysec
aof-use-rdb-preamble yes
save ""

# Security
aclfile /etc/redis/acl.conf
rename-command FLUSHALL ""
rename-command FLUSHDB ""

# Performance
hz 20
tcp-keepalive 300
timeout 300

# Slow log
slowlog-log-slower-than 10000
slowlog-max-len 256
latency-monitor-threshold 100

# Replication safety
min-replicas-to-write 1
min-replicas-max-lag 10
```

### 8.4 maxmemory & Eviction

- **maxmemory**: `2GB`
- **eviction policy**: `volatile-lru`
  - Cache keys: có TTL → có thể bị evict
  - Session/stream/idempotency: có TTL dài → ít bị evict hơn
  - Lock keys: TTL 10s → bị evict OK (TTL tự giải quyết)
  
**Alert khi**: `used_memory > 1.4GB` (70% of 2GB).

### 8.5 Failure Scenario & Recovery

```
Scenario: Primary Redis down lúc 8h sáng

T=0s:   Primary crash
T=5s:   Sentinel detect (down-after-milliseconds 5000)
T=10s:  Sentinel bầu quorum, chọn Replica 1 làm Primary mới
T=15s:  Replica 1 được promote
T=20s:  App (IOredis) tự động reconnect đến Primary mới qua Sentinel

Mất mát dữ liệu: Các write trong 5-20 giây trước crash (chưa sync sang replica)
App behavior:
  - Session: User có thể phải login lại
  - Lock: Tự expire → check-in duplicate được bảo vệ bởi DB UNIQUE constraint
  - Idempotency: Một số idempotency key bị mất → client retry → DB constraint bắt
  - Cache: Cache miss → query DB → hot path vẫn hoạt động
  
Recovery: Tự động qua Sentinel, không cần manual intervention
```

### 8.6 Metrics Alert Configuration

```yaml
# Prometheus alerts
- name: Redis Memory High
  condition: used_memory / maxmemory > 0.7
  severity: warning

- name: Redis Memory Critical
  condition: used_memory / maxmemory > 0.85
  severity: critical

- name: Redis Eviction Detected
  condition: rate(evicted_keys[5m]) > 0
  severity: warning

- name: Redis Replication Lag High
  condition: replication_lag > 5
  severity: critical

- name: Redis Hit Ratio Low
  condition: keyspace_hits / (keyspace_hits + keyspace_misses) < 0.8
  severity: warning

- name: Redis Slow Commands
  condition: slowlog_length > 10
  severity: warning

- name: Redis Rejected Connections
  condition: rejected_connections > 0
  severity: critical
```

### 8.7 ACL Final Design

```bash
# App user
ACL SETUSER timekeeping-app on >AppSecureP@ss2026 \
  ~cache:* ~session:* ~otp:* ~rate:* ~lock:* \
  ~stats:* ~online:* ~leaderboard:* ~attendance:* \
  ~idempotency:* ~stream:attendance-events \
  +@read +@write +@string +@hash +@set +@sortedset +@bitmap \
  +eval +evalsha +xadd +xread +xgroup +xreadgroup +xack \
  +xpending +xautoclaim +expire +persist +ttl +exists +del +unlink \
  -@dangerous -@admin -config -flushall -flushdb -keys -debug

# Stream worker user
ACL SETUSER stream-worker on >WorkerSecureP@ss \
  ~stream:* \
  +xreadgroup +xack +xpending +xautoclaim +xclaim +xadd +xdel

# Monitoring user (read-only)
ACL SETUSER prometheus-monitor on >MonitorP@ss \
  ~* +info +dbsize +slowlog|get +slowlog|len +client|list \
  +latency|latest +latency|history +memory|usage +memory|doctor

# Disable default user
ACL SETUSER default off
```

---

## 9. Checklist Day 4

### Kiến thức

- [ ] Giải thích RDB và AOF, ưu/nhược điểm mỗi loại
- [ ] Chọn persistence mode phù hợp cho từng use case
- [ ] Giải thích maxmemory và cách tính maxmemory phù hợp
- [ ] Nói được sự khác biệt 4 eviction policy quan trọng
- [ ] Mô tả replication lag và hậu quả khi Primary crash
- [ ] Giải thích Sentinel là gì và tại sao cần 3 node
- [ ] Giải thích hash slot và hash tag trong Cluster
- [ ] Đọc được output `INFO memory`, `INFO stats`, `INFO replication`
- [ ] Tính được hit ratio và biết ngưỡng target
- [ ] Giải thích ACL và tại sao cần least privilege

### Thực hành

- [ ] Chạy Redis với persistence (AOF everysec)
- [ ] Cấu hình maxmemory và test eviction
- [ ] Setup replication (1 primary + 1 replica) bằng Docker
- [ ] Chạy SLOWLOG và phân tích kết quả
- [ ] Chạy LATENCY DOCTOR
- [ ] Tạo ACL user với quyền giới hạn
- [ ] Test AUTH với ACL user
- [ ] Đọc INFO và tìm các metric quan trọng

---

## 10. Senior Review Questions Day 4

**Q1**: Tại sao cần cả RDB lẫn AOF? Không dùng 1 cái được không?

> **Answer**: AOF everysec đảm bảo mất tối đa 1s data, nhưng file lớn và restart chậm (replay toàn bộ log). RDB restart nhanh nhưng mất nhiều data giữa 2 snapshot. Redis 4.0+ hybrid mode (aof-use-rdb-preamble yes) giải quyết: AOF file bắt đầu bằng RDB snapshot, rồi append AOF commands → restart nhanh (load RDB phần đầu) + ít mất data (AOF phần sau). Production nên dùng hybrid.

**Q2**: volatile-lru vs allkeys-lru — chọn cái nào cho timekeeping?

> **Answer**: `volatile-lru` phù hợp hơn cho timekeeping vì: Cache keys có TTL → có thể bị evict. Session, lock, idempotency keys cũng có TTL nhưng dài hơn → ít bị evict hơn cache. `allkeys-lru` risk evict session/lock là những key quan trọng hơn. Tuy nhiên, nếu dùng 2 Redis instance (cache và state), dùng `allkeys-lru` cho cache instance và `noeviction` cho state instance là tối ưu nhất.

**Q3**: Sentinel dùng cho HA nhưng Cluster cũng có HA. Khi nào chọn Sentinel, khi nào chọn Cluster?

> **Answer**: Sentinel: HA cho single shard, vertical scale, multi-key operations đầy đủ, đơn giản hơn. Dùng khi data < 10GB và 1 primary đủ. Cluster: Horizontal sharding, scale out multiple primaries, nhưng giới hạn multi-key operations (phải cùng slot/hash tag), phức tạp hơn. Dùng khi cần > 10GB data, write throughput > 100k ops/s, hoặc cần scale out. Timekeeping system với < 100MB data → Sentinel là đủ.

**Q4**: mem_fragmentation_ratio = 2.5. Điều gì xảy ra và phải làm gì?

> **Answer**: Fragmentation 2.5 có nghĩa Redis đang yêu cầu OS cấp phát 2.5x so với thực sự dùng. Nguyên nhân: Nhiều key bị delete/expire → fragmented memory blocks. Xử lý: `MEMORY PURGE` (Redis 4.0+) để de-fragment. Hoặc restart Redis (trigger RDB load lại → clean memory). Monitor: Nếu sau MEMORY PURGE vẫn cao → có thể do pattern sử dụng (nhiều key nhỏ, thay đổi liên tục). Xem xét key design.

**Q5**: App đang dùng connection pool size = 50 per instance, 10 instances. Redis maxclients = 10000. Có vấn đề không?

> **Answer**: 50 × 10 = 500 connections. maxclients = 10000 → OK về số lượng. Nhưng ioredis (và hầu hết Redis clients hiện đại) dùng 1 connection per instance và multiplex tất cả request qua đó. Nếu đang tạo 50 connections per instance, có thể là code đang tạo nhiều Redis instances (anti-pattern). Kiểm tra lại: mỗi module/service nên inject 1 singleton Redis client, không tạo mới. 500 connections × 10KB overhead = 5MB — chấp nhận được nhưng vẫn nên optimize.

**Q6**: Failover xảy ra, app mất 15 giây. Trong 15 giây đó, những gì bị ảnh hưởng trong timekeeping?

> **Answer**: (1) Check-in: Lock expire → check-in bị duplicate → DB UNIQUE constraint bắt → insert bị fail → checkin API trả lời error thay vì 200 → UX xấu nhưng data OK. (2) Session: User đang request có thể gặp error → retry tự động hoặc phải login lại. (3) Cache: Miss → fallback tới DB → DB tải tăng tạm thời → cần DB có đủ capacity. (4) Rate limit: Counter bị mất → window reset → user có thể gửi nhiều request hơn trong 15 giây đó. (5) Stream: XADD fail → attendance event bị mất → worker không nhận được → cần monitor và alert. Tổng kết: Failover 15 giây là chấp nhận được với design đúng (DB constraint + retry + fallback). Quan trọng là app code phải xử lý Redis connection error gracefully, không crash.

---

## Appendix: Production Redis Checklist

### Trước Khi Deploy

- [ ] `maxmemory` đặt phù hợp (< 60% RAM)
- [ ] `maxmemory-policy` đã chọn đúng
- [ ] `protected-mode yes`
- [ ] `bind` chỉ trên private interface
- [ ] ACL đã cấu hình, default user disabled
- [ ] `FLUSHALL` / `FLUSHDB` đã rename hoặc disable
- [ ] Persistence phù hợp cho use case
- [ ] `slowlog-log-slower-than` đã bật
- [ ] `latency-monitor-threshold` đã bật

### Monitoring

- [ ] Prometheus Redis Exporter đã chạy
- [ ] Alert trên memory (70%, 85%)
- [ ] Alert trên eviction
- [ ] Alert trên replication lag
- [ ] Alert trên hit ratio (< 80%)
- [ ] Alert trên rejected_connections
- [ ] SLOWLOG review hàng tuần

### Hạ Tầng

- [ ] Redis không trên cùng server với DB
- [ ] Network latency Redis < 1ms
- [ ] SSD cho persistence
- [ ] Sentinel hoặc Cluster cho production
- [ ] Backup RDB định kỳ (dù có AOF)
- [ ] Runbook cho failover

### Application

- [ ] Redis client dùng singleton pattern
- [ ] Retry logic cho Redis error
- [ ] Fallback khi Redis down (graceful degradation)
- [ ] Không lưu sensitive data không mã hóa
- [ ] TTL có trên mỗi cache key

---

*Day 4 hoàn thành. Redis Production series kết thúc.*
*Next: Áp dụng vào Timekeeping System — implement từng layer.*
