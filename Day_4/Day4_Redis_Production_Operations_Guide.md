# Day 4: Production Redis — Operations Guide

> **Mindset**: Day 4 la su khac biet giua "Redis chay duoc" va "Redis chay dung trong production". Mot Redis cau hinh sai co the mat data, bi exploit, hoac down het cluster luc cao diem.

---

## Muc Tieu

Sau Day 4, ban phai:

- [ ] Giai thich RDB vs AOF va chon dung cho tung use case
- [ ] Config maxmemory va eviction policy phu hop
- [ ] Mo ta replication lag va hau qua cua no
- [ ] Giai thich Sentinel vs Cluster — khi nao dung cai nao
- [ ] Doc va hieu output cua `INFO`, `SLOWLOG`, `LATENCY DOCTOR`
- [ ] Tinh duoc hit ratio va biet nguong canh bao
- [ ] Setup ACL dung voi least privilege
- [ ] Thiet ke production Redis cho timekeeping system

**Context Project**: Timekeeping System. 500-2000 employees. Peak load 8:00-8:30 AM. Critical data trong PostgreSQL, Redis la cache/coordination layer.

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
Redis ACL AUTH ACL SETUSER ACL WHOAMI
Redis TLS protected-mode requirepass
Redis production checklist
Redis managed cloud: ElastiCache Upstash Redis Cloud
```

---

## 1. Persistence

### 1.1 RDB (Redis Database Snapshot)

RDB = Periodic fork va dump toan bo data ra file nhi phan.

```
Redis RAM → fork() → child process → serialize → /data/dump.rdb
```

**Cau hinh:**

```conf
# redis.conf
# Snapshot neu >= 1 key thay doi trong 3600 giay (1 gio)
save 3600 1
# Snapshot neu >= 100 key thay doi trong 300 giay (5 phut)
save 300 100
# Snapshot neu >= 10000 key thay doi trong 60 giay (1 phut)
save 60 10000

# Tat RDB
save ""

# File path
dir /data
dbfilename dump.rdb

# Neu RDB fail → tu choi write (an toan hon)
stop-writes-on-bgsave-error yes

# Compress RDB file (CPU vs disk trade-off)
rdbcompression yes
```

**Manual trigger:**

```bash
BGSAVE          # Background snapshot (non-blocking)
SAVE            # Foreground snapshot (blocking — khong dung production)
LASTSAVE        # Unix timestamp cua lan snapshot cuoi
```

**Dac diem:**

| Tieu chi | RDB |
|---|---|
| Data mat toi da | Giua 2 lan snapshot (co the 1-60 phut) |
| Toc do restart | Nhanh (load binary file) |
| Kich thuoc file | Nho, compact |
| CPU anh huong | fork() tao spike nho (co the vai tram ms) |
| Use case | Backup, disaster recovery |

---

### 1.2 AOF (Append-Only File)

AOF = Ghi moi write command vao file log. Khi restart, replay lai toan bo log.

```
SET key val   →  [*3\r\n$3\r\nSET\r\n$3\r\nkey\r\n$3\r\nval\r\n] → appendonly.aof
INCR counter  →  [ghi vao cuoi file]
DEL key       →  [ghi vao cuoi file]
```

**Cau hinh:**

```conf
# redis.conf
appendonly yes
appendfilename "appendonly.aof"
dir /data

# Fsync strategy — QUAN TRONG NHAT
appendfsync always      # Flush moi command, fsync ngay → an toan nhat, CHAM nhat (~1000 ops/s)
appendfsync everysec    # Flush + fsync moi giay → BALANCE tot (mat toi da 1 giay data)
appendfsync no          # De OS quyet dinh (thuong 30s) → NHANH nhat, mat nhieu nhat

# AOF rewrite (compact lai file lon)
auto-aof-rewrite-percentage 100    # Rewrite khi AOF lon gap doi baseline
auto-aof-rewrite-min-size 64mb     # Chi rewrite neu file > 64MB

# An toan khi AOF bi truncated (crash giua khi ghi)
aof-use-rdb-preamble yes           # Hybrid: RDB snapshot + AOF delta (Redis 4.0+)
```

**Manual rewrite:**

```bash
BGREWRITEAOF    # Compact AOF file (background, non-blocking)
```

**Dac diem:**

| Tieu chi | AOF everysec | AOF always | AOF no |
|---|---|---|---|
| Data mat toi da | 1 giay | 0 (gan nhu) | Nhieu giay |
| Toc do ghi | Cao | Thap (~1k ops/s) | Cao nhat |
| Kich thuoc file | Lon | Lon | Lon |
| Restart speed | Cham (replay log) | Cham | Cham |

---

### 1.3 No Persistence

```conf
save ""
appendonly no
```

Restart → mat toan bo. Dung cho **pure cache** — data lay lai tu PostgreSQL khi miss.

---

### 1.4 Nen Chon Gi? (Decision Matrix)

| Use Case | Persistence | Ly Do |
|---|---|---|
| Employee profile cache | **None** | DB la source of truth, cache miss OK |
| Attendance summary cache | **None** | Rebuild duoc tu DB |
| Session tokens | **AOF everysec** | Mat session → user bi kick → bad UX |
| OTP codes | **AOF everysec** | Mat OTP → user khong login duoc |
| Distributed locks | **None / AOF everysec** | Lock ngan (10s), mat lock = TTL expire tu giai quyet |
| Rate limit counters | **None** | Mat counter = window reset = chap nhan duoc |
| Idempotency keys | **AOF everysec** | Mat key → duplicate write → nguy hiem |
| Stream events | **AOF everysec** | Mat event = mat du lieu nghiep vu |

**Timekeeping recommendation:**

```conf
# Single Redis instance cho moi thu
appendonly yes
appendfsync everysec
save ""          # Tat RDB, dung AOF thuan
aof-use-rdb-preamble yes  # Hybrid mode
```

**Hoac** dung 2 Redis instance:
- `redis-cache`: No persistence, allkeys-lru (pure cache)
- `redis-state`: AOF everysec, noeviction (session/lock/stream)

---

### 1.5 RDB + AOF Hybrid (Redis 4.0+)

```conf
aof-use-rdb-preamble yes
```

File AOF bat dau bang RDB snapshot → append AOF commands tu do tro di. Restart nhanh hon AOF thuan, du lieu day du hon RDB thuan.

---

## 2. Memory & Eviction

### 2.1 maxmemory

```conf
# Dat gioi han memory
maxmemory 2gb
maxmemory 512mb
maxmemory 0       # Khong gioi han (nguy hiem tren server co limit)
```

**Uoc tinh memory can thiet:**

```
Timekeeping system:
- 2000 employees × 1KB per profile cache = 2MB
- 2000 sessions × 500B per session = 1MB
- Streams: 50000 messages × 200B per message = 10MB
- Leaderboard, counters, locks: ~5MB
- Redis overhead: ~30MB

Tong: ~50MB voi normal load
→ Dat maxmemory = 256mb (buffer nhieu)
→ Alert khi > 70% (180MB)
→ Check khi > 85% (218MB)
```

```bash
# Kiem tra hien tai
INFO memory

# Output quan trong:
# used_memory:           50331648   (50MB dang dung)
# used_memory_peak:      78643200   (78MB peak)
# maxmemory:             268435456  (256MB gioi han)
# mem_fragmentation_ratio: 1.2      (OK, < 1.5 la binh thuong)
```

**mem_fragmentation_ratio:**
- `1.0 - 1.5`: Binh thuong
- `> 1.5`: Memory fragmentation — Redis dang dung nhieu hon can
- `< 1.0`: Redis dang swap ra disk — NGUY HIEM, latency tang vot

### 2.2 Eviction Policies

Khi Redis dat `maxmemory`, no phai quyet dinh xoa key nao:

```conf
maxmemory-policy allkeys-lru
```

| Policy | Mo Ta | Dung Khi |
|---|---|---|
| `noeviction` | Tu choi write, tra loi OOM error | Khong muon mat data, se alert va xu ly thu cong |
| `allkeys-lru` | Xoa key it su dung nhat (LRU) trong tat ca key | Pure cache, moi key co the bi xoa |
| `allkeys-lfu` | Xoa key it truy cap nhat (LFU) trong tat ca key | Cache voi access pattern khong deu |
| `allkeys-random` | Xoa key ngau nhien | Khong nen dung |
| `volatile-lru` | LRU trong cac key co TTL | Mixed: co key can giu, co key co the xoa |
| `volatile-lfu` | LFU trong cac key co TTL | Mixed voi LFU |
| `volatile-ttl` | Xoa key co TTL ngan nhat truoc | Muon xoa key sap het han |
| `volatile-random` | Random trong key co TTL | Hiem khi dung |

**LRU vs LFU:**
- **LRU** (Least Recently Used): Xoa key **lau nhat chua duoc truy cap**. Phu hop voi temporal access pattern.
- **LFU** (Least Frequently Used): Xoa key **it duoc truy cap nhat**. Phu hop voi hot/cold data distribution.

**Timekeeping recommendation:**

```conf
# Redis cache instance
maxmemory-policy allkeys-lru

# Redis state instance (session, lock, stream)
maxmemory-policy noeviction   # Khong tu dong xoa, alert khi day
```

**Luu y**: Voi `noeviction`, khi day, Redis tra loi `OOM command not allowed` cho moi write. Phai monitor va xu ly truoc khi day.

---

## 3. Replication

### 3.1 Primary-Replica

```
       WRITE                     READ (optional)
Client ─────→ Primary ─────────→ Replica 1
                     └─────────→ Replica 2
                     
Replication: ASYNC (default)
```

**Cau hinh Primary:**

```conf
# primary khong can them config dac biet
# Chi can replica biet dia chi primary
bind 0.0.0.0
protected-mode yes
requirepass "strong-password-here"
```

**Cau hinh Replica:**

```conf
replicaof 192.168.1.10 6379
masterauth "strong-password-here"
replica-read-only yes    # Replica chi cho phep READ
```

**Kiem tra replication:**

```bash
# Tren primary
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

**Replication la ASYNC** — replica co the lag sau primary.

```
Primary: WRITE → SET key 100         (offset: 1000)
Replica: SYNC  ... (lag 50ms) ...    (offset: 950)

If client reads from replica: co the thay gia tri cu
```

**Hau qua thuc te:**
- Client doc tu replica: co the thay du lieu cu
- Failover: neu primary crash, replica chua sync → mat cac write gan nhat

**Kiem tra lag:**

```bash
# Tren primary
INFO replication | grep lag
# slave0:ip=...,lag=0   → Dong bo tot
# slave0:ip=...,lag=5   → Lag 5 giay → can kiem tra

# Thiet lap threshold alert
redis.conf: min-slaves-max-lag 10
# Primary tu choi write neu tat ca replica lag > 10 giay
```

**Xu ly Replication Lag trong App:**

```typescript
// Strategy 1: Luon doc tu primary (don gian, tot cho consistency)
// Dung primary cho ca READ va WRITE

// Strategy 2: Read-your-own-writes
// Sau write → doc tu primary (hoac doi replica sync)
// Voi IOredis:
const primaryClient = new Redis({ host: 'primary', port: 6379 });
const replicaClient = new Redis({ host: 'replica', port: 6379 });

// Chi dung replica cho data khong critical
async getAttendanceSummary(empId: number, date: string) {
  // Stale OK → read from replica
  return replicaClient.get(`cache:attendance-summary:${empId}:${date}`);
}

async getSessionData(token: string) {
  // Session phai chinh xac → read from primary
  return primaryClient.get(`session:${token}`);
}
```

---

### 3.3 Sentinel — High Availability

Sentinel giai quyet: **Primary down → tu dong failover sang Replica**.

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

**Yeu cau toi thieu**: 3 Sentinel (de bau ra quorum majority = 2/3).

**Cau hinh Sentinel:**

```conf
# sentinel.conf
sentinel monitor mymaster 192.168.1.10 6379 2
# mymaster = ten (dat tuy y)
# 192.168.1.10 6379 = primary address
# 2 = so sentinel dong y de failover (quorum)

sentinel auth-pass mymaster "strong-password"
sentinel down-after-milliseconds mymaster 5000     # 5s khong pong → subjectively down
sentinel failover-timeout mymaster 60000           # Timeout failover sau 60s
sentinel parallel-syncs mymaster 1                 # Moi luc 1 replica sync (de giam tai)
```

**App kết noi Sentinel:**

```typescript
// ioredis voi Sentinel
import Redis from 'ioredis';

const redis = new Redis({
  sentinels: [
    { host: '192.168.1.20', port: 26379 },
    { host: '192.168.1.21', port: 26379 },
    { host: '192.168.1.22', port: 26379 },
  ],
  name: 'mymaster',        // Ten khai bao trong sentinel.conf
  password: 'strong-password',
  sentinelPassword: 'sentinel-password',
  retryStrategy: (times) => Math.min(times * 50, 2000), // Retry voi backoff
});

redis.on('error', (err) => console.error('Redis error:', err));
redis.on('reconnecting', () => console.log('Redis reconnecting...'));
```

**Sentinel gioi han:**
- Sentinel la **HA, khong phai sharding**
- Van chi 1 primary cho write → vertical scale only
- Failover mat vai giay → app phai xu ly retry

---

## 4. Redis Cluster

### 4.1 Hash Slots

Redis Cluster phan tan data qua **16384 hash slots**.

```
HASH_SLOT = CRC16(key) % 16384

Cluster 3 primary nodes:
Node A: slots 0     - 5460     (node 192.168.1.10)
Node B: slots 5461  - 10922    (node 192.168.1.11)
Node C: slots 10923 - 16383    (node 192.168.1.12)

Key "employee:100" → CRC16("employee:100") % 16384 = 7638 → Node B
Key "session:abc"  → CRC16("session:abc") % 16384  = 1234 → Node A
```

**MOVED redirect**: Neu client goi sai node, Redis tra loi `MOVED`:

```
Client → Node A: GET employee:100
Node A: -MOVED 7638 192.168.1.11:6379
Client → Node B: GET employee:100 (redirect)
```

Smart Redis clients (ioredis) tu xu ly MOVED redirect.

### 4.2 Hash Tag

Neu can 2 key cung nam 1 slot (de dung multi-key command), dung **hash tag**: phan trong `{}` duoc dung de tinh hash.

```bash
# Khong co hash tag: khac slot → khong the dung MGET/MULTI trong cluster
employee:100:profile   → CRC16("employee:100:profile") → slot A
employee:100:session   → CRC16("employee:100:session") → slot B

# Co hash tag: cung slot → co the dung MGET/MULTI
user:{100}:profile     → CRC16("100") → slot X
user:{100}:session     → CRC16("100") → slot X (cung slot!)

# Thuc hanh voi timekeeping
cache:{emp:100}:profile
cache:{emp:100}:schedule
cache:{emp:100}:attendance
```

```bash
# Voi hash tag cung slot → MGET hop le trong cluster
MGET user:{100}:profile user:{100}:session  # OK
MGET employee:100 session:100               # ERROR (khac slot)
```

### 4.3 Multi-key Limitation trong Cluster

```bash
# CAC LENH KHONG HOAT DONG CROSS-SLOT
MGET key1 key2          # Chi OK neu cung slot
MSET key1 val key2 val  # Chi OK neu cung slot
SUNIONSTORE dest k1 k2  # Chi OK neu cung slot
MULTI
  INCR key1             # Cross-slot in MULTI → ERROR
  SET key2 val
EXEC
```

**Giai phap:**

```typescript
// Thay vi MGET cross-slot, dung pipeline individual GETs
const pipeline = redis.pipeline();
ids.forEach(id => pipeline.get(`cache:employee:${id}`));
const results = await pipeline.exec();
// Pipeline hoat dong tot trong cluster (moi command routing rieng)
```

### 4.4 Cluster vs Sentinel vs Standalone

| Tieu chi | Standalone | Sentinel | Cluster |
|---|---|---|---|
| HA (High Availability) | Khong | Co | Co |
| Horizontal sharding | Khong | Khong | Co |
| Multi-key commands | Day du | Day du | Gioi han (same slot) |
| Complexity | Thap | Trung binh | Cao |
| Scale-out | Khong | Khong | Co |
| Khi dung | Dev, small prod | Prod, single shard | Prod, nhieu GB data |

**Timekeeping recommendation**: **Sentinel** tren 2-3 server. Data khong lon (< 1GB), Cluster them phuc tap khong can thiet. Hoac dung **managed Redis** (ElastiCache, Upstash) de tranh operational burden.

---

## 5. Monitoring

### 5.1 INFO Sections

```bash
INFO                    # Tat ca sections
INFO server             # Version, OS, port, uptime
INFO clients            # So ket noi, blocked clients
INFO memory             # Memory usage, fragmentation
INFO persistence        # RDB/AOF status
INFO stats              # Command count, hit/miss, eviction
INFO replication        # Primary/replica status, lag
INFO keyspace           # So key per DB, TTL info
```

**Cac metric quan trong can theo doi:**

```bash
INFO memory
# used_memory:                50331648     # Dang dung 50MB
# used_memory_rss:            67108864     # OS thay 64MB (RSS > used = fragmentation)
# mem_fragmentation_ratio:    1.33         # OK (< 1.5)
# mem_fragmentation_bytes:    12345678     # So byte bi fragment

INFO stats
# instantaneous_ops_per_sec:  12345        # Hien tai 12k ops/s
# total_commands_processed:   987654321    # Tong so command tu khi khoi dong
# keyspace_hits:              900000       # 900k cache hits
# keyspace_misses:            100000       # 100k cache misses
# expired_keys:               5000         # So key da expire
# evicted_keys:               0            # So key bi evict (CANH BAO neu > 0 lien tuc)
# rejected_connections:       0            # So ket noi bi tu choi (CANH BAO neu > 0)

INFO clients
# connected_clients:          150          # Hien tai 150 connection
# blocked_clients:            0            # Khong co client bi block
# maxclients:                 10000        # Gioi han ket noi

INFO replication
# role:master
# connected_slaves:           2
# slave0:ip=...,lag:          0            # CANH BAO neu lag > 5s
```

### 5.2 Hit Ratio

```
Hit Ratio = keyspace_hits / (keyspace_hits + keyspace_misses)
```

```bash
# Lay tu INFO stats
redis-cli info stats | grep -E "keyspace_hits|keyspace_misses"

# Tinh toan
hits = 900000
misses = 100000
hit_ratio = 900000 / (900000 + 100000) = 0.90 = 90%
```

| Hit Ratio | Danh Gia | Hanh Dong |
|---|---|---|
| > 90% | Tot | Binh thuong |
| 80-90% | Trung binh | Xem lai TTL, cache strategy |
| 70-80% | Thap | Review key naming, data type |
| < 70% | Kem | Cache ko hieu qua, review architecture |

**Timekeeping target**: > 85% vao binh thuong, > 75% vao peak 8h sang (nhieu cache miss do check-in moi).

### 5.3 SLOWLOG

```bash
# Cau hinh
redis.conf:
slowlog-log-slower-than 10000   # Log command cham hon 10ms (10000 microseconds)
slowlog-max-len 128              # Giu toi da 128 entry

# Xem slow log
SLOWLOG GET 10          # 10 entries gan nhat
SLOWLOG GET             # Tat ca
SLOWLOG LEN             # So entry hien tai
SLOWLOG RESET           # Xoa slowlog

# Format output:
# 1) (integer) 14              ← ID
# 2) (integer) 1720401600      ← Unix timestamp
# 3) (integer) 15234           ← Thoi gian thuc thi (microseconds)
# 4) 1) "KEYS"                 ← Command
#    2) "*"                    ← Arguments ← DAY CHINH LA VAN DE
# 5) "127.0.0.1:54321"        ← Client
# 6) ""                        ← Client name
```

**Phan tich SLOWLOG:**
- `KEYS *` → Thay bang `SCAN`
- `SMEMBERS large-set` → Dung `SSCAN`
- `HGETALL large-hash` → Giu hash nho, dung `HMGET` field cu the
- `ZRANGE 0 -1` tren large ZSet → Them `COUNT` limit

### 5.4 LATENCY

```bash
# Bat latency monitoring
redis.conf:
latency-monitor-threshold 100    # Log event tre hon 100ms

# Commands
LATENCY LATEST              # Cac event tre gan nhat
LATENCY HISTORY event-name  # Lich su cua event cu the
LATENCY RESET               # Reset tat ca
LATENCY DOCTOR              # Phan tich va goi y tu dong (rat huu ich)

# LATENCY DOCTOR output example:
# I detected 2 latency samples:
# 1. fork: 354ms. ADVICE: Consider lowering rdb-save-incremental-fsync to yes
# 2. aof_write_pending_fsync: 12ms. ADVICE: Use appendfsync everysec instead of always
```

### 5.5 CLIENT LIST

```bash
CLIENT LIST                     # Tat ca client dang ket noi
CLIENT LIST TYPE normal         # Loai: normal, pubsub, slave, monitor
CLIENT KILL ID 12345            # Ngat ket noi client cu the
CLIENT SETNAME myapp-worker-1   # Dat ten cho connection (debug de hon)
CLIENT GETNAME                  # Lay ten hien tai

# Client LIST output:
# id=5 addr=127.0.0.1:54321 fd=12 name=myapp-worker cmd=brpop age=120 idle=0 ...
# id=6 addr=127.0.0.1:54322 fd=13 name= cmd=subscribe age=3600 idle=3600 ...
```

### 5.6 Dashboard Monitoring

**Metrics can alert:**

| Metric | Nguong Canh Bao | Nguong Critical | Hanh Dong |
|---|---|---|---|
| `used_memory` | > 70% maxmemory | > 85% maxmemory | Tang maxmemory hoac xoa key |
| `mem_fragmentation_ratio` | > 1.5 | > 2.0 | MEMORY PURGE hoac restart |
| `evicted_keys` | > 0 (lien tuc) | > 100/phut | Tang maxmemory hoac review TTL |
| `rejected_connections` | > 0 | > 0 | Tang maxclients, check connection pool |
| `blocked_clients` | > 10 | > 50 | Xem BLPOP/BRPOP timeout |
| `replication lag` | > 2s | > 10s | Kiem tra mang, xem BGSAVE |
| `slowlog entries` | > 10/phut | > 50/phut | Xem SLOWLOG, tim command loi |
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

Cac dashboard co san: `grafana.com/dashboards/11835` (Redis Sentinel) hoac `grafana.com/dashboards/763`.

---

## 6. Security

### 6.1 Network Security — Quan Trong Nhat

```conf
# TUYET DOI KHONG bind 0.0.0.0 neu expose ra internet
bind 127.0.0.1 192.168.1.10    # Chi nghe tren interface noi bo
protected-mode yes              # Tu dong bat neu khong co bind va password
port 6379

# Doi port mac dinh (security through obscurity, it hieu qua nhung them 1 lop)
port 6380

# Disable cac command nguy hiem
rename-command FLUSHALL ""      # Disable FLUSHALL
rename-command FLUSHDB ""       # Disable FLUSHDB
rename-command DEBUG ""         # Disable DEBUG
rename-command CONFIG "CONFIG_SECURE_KEY_abc123"  # Doi ten CONFIG
```

### 6.2 Authentication

```conf
# Legacy: requirepass (tat ca user cung password)
requirepass "VeryStr0ng-P@ssw0rd-2026!"

# Modern: ACL (Redis 6+)
aclfile /etc/redis/acl.conf

# Tat default user
ACL SETUSER default off
```

### 6.3 ACL (Access Control List)

ACL cho phep tao user voi quyen cu the.

```bash
# Xem user hien tai
ACL LIST
ACL WHOAMI
ACL CAT         # List tat ca command categories
ACL CAT string  # List tat ca command trong category string

# Tao user cho app (chi duoc dung key cache:*)
ACL SETUSER app-user on >AppStr0ngP@ss ~cache:* +get +set +del +expire +exists +ttl +type
# on         = user active
# >password  = password
# ~cache:*   = chi duoc truy cap key bat dau bang cache:
# +get +set  = chi duoc dung cac command nay

# User doc toa bo (chi GET, khong write)
ACL SETUSER readonly-user on >ReadOnlyP@ss ~* +get +mget +hget +hmget +hgetall +lrange +smembers +zrange

# User full quyền cho ops team (voi gioi han key khong qua rong)
ACL SETUSER ops-admin on >OpsP@ss ~* +@all

# Save ACL ra file
ACL SAVE

# Load lai ACL tu file
ACL LOAD
```

**ACL cho Timekeeping System:**

```bash
# App service user: full CRUD tren namespace cua no
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

# Worker service user: chi stream
ACL SETUSER stream-worker on >WorkerP@ss \
  ~stream:* \
  +xreadgroup +xack +xpending +xautoclaim +xclaim +xadd

# Monitoring user: chi read-only
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
tls-auth-clients yes   # Yeu cau client certificate
```

```typescript
// ioredis voi TLS
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

- [ ] Khong expose Redis ra public internet
- [ ] Bind tren private interface only
- [ ] `protected-mode yes`
- [ ] Dung ACL, disable default user
- [ ] Password manh (>20 chars)
- [ ] TLS trong cloud/multi-datacenter
- [ ] Disable FLUSHALL, FLUSHDB (rename-command)
- [ ] Khong log sensitive data (OTP, session token) trong app logs
- [ ] Network firewall: chi allow app servers truy cap Redis port
- [ ] Monitor connected_clients bat thuong (co the la intrusion)

---

## 7. Operational Pitfalls

### P1: Fork Lag trong BGSAVE / BGREWRITEAOF

```
Redis dung fork() de BGSAVE
fork() phai copy page table cua process hien tai
Neu Redis dang dung 2GB → fork() mat 1-2 giay → tat ca command bi block
```

**Giai phap:**
- Cau hinh: `no-appendfsync-on-rewrite yes` — tranh AOF fsync trong khi rewrite
- Dat BGSAVE vao gio it tai
- Dung smaller Redis instances thay vi 1 instance lon

---

### P2: Maxmemory Dat Tren Tong RAM

```
Server co 4GB RAM
maxmemory = 0 (khong gioi han) → Redis dung het → OS swap → latency vot
```

**Rule**: `maxmemory <= 50-60% total RAM` de danh cho OS va process khac.

---

### P3: Replication Full Resync

```
Replica bi ngat ket noi > repl-backlog-size / toc do write
→ Redis phai FULL RESYNC (dump lai toan bo data)
→ Trong thoi gian do: CPU/disk/memory spike o Primary
```

**Giai phap:**

```conf
# Tang repl-backlog
repl-backlog-size 512mb   # Mac dinh 1MB → tang len neu replica hay bi ngat
repl-timeout 60           # Timeout cho replication connection
```

---

### P4: Cluster Resharding Lam Gian Doan

Khi them node vao cluster, phai resharding (di chuyen hash slot). Trong qua trinh do, mot so key khong truy cap duoc.

**Giai phap**: Resharding vao gio low traffic. Dung `redis-cli --cluster reshard` voi `--cluster-pipeline` de tang toc. Lam tung batch nho.

---

### P5: Khong Monitor Slowlog

`KEYS *` bi chay trong production (code cu, debug script) → block 500ms → tat ca request treo → timeout → incident.

**Giai phap**: Dat alert khi SLOWLOG co entry > 10ms. Review code truoc deploy: tim KEYS, SMEMBERS, HGETALL.

---

### P6: Connection Pool Qua Lon

```
10 NestJS instances × 100 Redis connections = 1000 connections
Redis mac dinh maxclients = 10000 → OK
Nhung moi connection ton ~10KB memory → 1000 connections = 10MB overhead
```

**Giai phap**: Ioredis su dung single connection per instance (multiplexed). Khong tao nhieu Redis instances trong code. Dung connection pool dung cach.

```typescript
// BAD: Tao moi Redis instance moi request
async getEmployee(id: number) {
  const redis = new Redis({ host: '...', port: 6379 }); // Moi lan tao moi!
  const result = await redis.get(`cache:employee:${id}`);
  await redis.quit();
  return result;
}

// GOOD: Dung singleton
@Module({
  providers: [{
    provide: 'REDIS_CLIENT',
    useFactory: () => new Redis({ host: '...', port: 6379 }),
  }]
})
```

---

### P7: Khong Dat TTL cho Cache Key

Memory tang theo thoi gian → eviction bat dau xoa key → cache hit ratio giam dot ngot → DB bi hit nang.

```bash
# Kiem tra key khong co TTL
SCAN 0 COUNT 100
# Voi moi key: TTL key → neu -1 thi khong co TTL

# Kiem tra nhanh
redis-cli --scan --pattern "cache:*" | xargs -L 1 redis-cli TTL | grep -c "^-1"
# Dem so cache key khong co TTL
```

---

## 8. Bai Tap Day 4: Thiet Ke Production Redis cho Timekeeping

### 8.1 Use Cases & Key Types

| Use Case | Keys | Type | TTL | Persistence Needed? |
|---|---|---|---|---|
| Employee profile cache | `cache:employee:{id}` | String (JSON) | 10 phut | Khong |
| Shift config cache | `cache:shift:{id}` | String (JSON) | 60 phut | Khong |
| Attendance summary cache | `cache:attendance:{id}:{date}` | String (JSON) | 2 phut | Khong |
| Session token | `session:{token}` | String | 8 gio | Co (AOF) |
| OTP code | `otp:{phone}` | String | 3 phut | Co (AOF) |
| OTP attempt | `otp:attempt:{phone}` | String (counter) | 15 phut | Khong |
| Login rate limit | `rate:login:{id}` | String (counter) | 15 phut | Khong |
| Check-in rate limit | `rate:checkin:{id}` | String (counter) | 1 phut | Khong |
| Distributed lock | `lock:{resource}` | String | 10 giay | Khong |
| Idempotency key | `idempotency:{path}:{key}` | String | 24 gio | Co (AOF) |
| Daily attendance bitmap | `attendance:{date}` | Bitmap | 25 gio | Khong |
| Online employees | `online:employees` | Set | No TTL | Khong |
| Check-in leaderboard | `leaderboard:early-checkin:{date}` | ZSet | 25 gio | Khong |
| Daily check-in counter | `stats:checkin:{date}` | String | 25 gio | Khong |
| Attendance stream | `stream:attendance-events` | Stream | MAXLEN ~50k | Co (AOF) |

### 8.2 Kien Truc De Xuat

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

### 8.3 Cau Hinh De Xuat

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
  - Cache keys: co TTL → co the bi evict
  - Session/stream/idempotency: co TTL dai → it bi evict hon
  - Lock keys: TTL 10s → bi evict OK (TTL tu giai quyet)
  
**Alert khi**: `used_memory > 1.4GB` (70% of 2GB).

### 8.5 Failure Scenario & Recovery

```
Scenario: Primary Redis down luc 8h sang

T=0s:   Primary crash
T=5s:   Sentinel detect (down-after-milliseconds 5000)
T=10s:  Sentinel bau quorum, chon Replica 1 lam Primary moi
T=15s:  Replica 1 duoc promote
T=20s:  App (IOredis) tu dong reconnect den Primary moi qua Sentinel

Mat mat du lieu: Cac write trong 5-20 giay truoc crash (chua sync sang replica)
App behavior:
  - Session: User co the phai login lai
  - Lock: Tu expire → check-in duplicate duoc bao ve boi DB UNIQUE constraint
  - Idempotency: Mot so idempotency key bi mat → client retry → DB constraint bat
  - Cache: Cache miss → query DB → hot path van hoat dong
  
Recovery: Tu dong qua Sentinel, khong can manual intervention
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

### Kien thuc

- [ ] Giai thich RDB va AOF, uu/nhuoc diem moi loai
- [ ] Chon persistence mode phu hop cho tung use case
- [ ] Giai thich maxmemory va cach tinh maxmemory phu hop
- [ ] Noi duoc su khac biet 4 eviction policy quan trong
- [ ] Mo ta replication lag va hau qua khi Primary crash
- [ ] Giai thich Sentinel la gi va tai sao can 3 node
- [ ] Giai thich hash slot va hash tag trong Cluster
- [ ] Doc duoc output `INFO memory`, `INFO stats`, `INFO replication`
- [ ] Tinh duoc hit ratio va biet nguong target
- [ ] Giai thich ACL va tai sao can least privilege

### Thuc hanh

- [ ] Chay Redis voi persistence (AOF everysec)
- [ ] Cau hinh maxmemory va test eviction
- [ ] Setup replication (1 primary + 1 replica) bang Docker
- [ ] Chay SLOWLOG va phan tich ket qua
- [ ] Chay LATENCY DOCTOR
- [ ] Tao ACL user voi quyen gioi han
- [ ] Test AUTH voi ACL user
- [ ] Doc INFO va tim cac metric quan trong

---

## 10. Senior Review Questions Day 4

**Q1**: Tai sao can ca RDB lan AOF? Khong dung 1 cai duoc khong?

> **Answer**: AOF everysec dam bao mat toi da 1s data, nhung file lon va restart cham (replay toan bo log). RDB restart nhanh nhung mat nhieu data giua 2 snapshot. Redis 4.0+ hybrid mode (aof-use-rdb-preamble yes) giai quyet: AOF file bat dau bang RDB snapshot, roi append AOF commands → restart nhanh (load RDB phan dau) + it mat data (AOF phan sau). Production nen dung hybrid.

**Q2**: volatile-lru vs allkeys-lru — chon cai nao cho timekeeping?

> **Answer**: `volatile-lru` phu hop hon cho timekeeping vi: Cache keys co TTL → co the bi evict. Session, lock, idempotency keys cung co TTL nhung dai hon → it bi evict hon cache. `allkeys-lru` risk evict session/lock la nhung key quan trong hon. Tuy nhien, neu dung 2 Redis instance (cache va state), dung `allkeys-lru` cho cache instance va `noeviction` cho state instance la toi uu nhat.

**Q3**: Sentinel dung cho HA nhung Cluster cung co HA. Khi nao chon Sentinel, khi nao chon Cluster?

> **Answer**: Sentinel: HA cho single shard, vertical scale, multi-key operations day du, don gian hon. Dung khi data < 10GB va 1 primary du. Cluster: Horizontal sharding, scale out multiple primaries, nhung gioi han multi-key operations (phai cung slot/hash tag), phuc tap hon. Dung khi can > 10GB data, write throughput > 100k ops/s, hoac can scale out. Timekeeping system voi < 100MB data → Sentinel la du.

**Q4**: mem_fragmentation_ratio = 2.5. Dieu gi xay ra va phai lam gi?

> **Answer**: Fragmentation 2.5 co nghia Redis dang yeu cau OS cap phat 2.5x so voi thuc su dung. Nguyen nhan: Nhieu key bi delete/expire → fragmented memory blocks. Xu ly: `MEMORY PURGE` (Redis 4.0+) de de-fragment. Hoac restart Redis (trigger RDB load lai → clean memory). Monitor: Neu sau MEMORY PURGE van cao → co the do pattern su dung (nhieu key nho, thay doi lien tuc). Xem xet key design.

**Q5**: App dang dung connection pool size = 50 per instance, 10 instances. Redis maxclients = 10000. Co van de khong?

> **Answer**: 50 × 10 = 500 connections. maxclients = 10000 → OK ve so luong. Nhung ioredis (va hau het Redis clients hien dai) dung 1 connection per instance va multiplex tat ca request qua do. Neu dang tao 50 connections per instance, co the la code dang tao nhieu Redis instances (anti-pattern). Kiem tra lai: moi module/service nen inject 1 singleton Redis client, khong tao moi. 500 connections × 10KB overhead = 5MB — chap nhan duoc nhung van nen optimize.

**Q6**: Failover xay ra, app mat 15 giay. Trong 15 giay do, nhung gi bi anh huong trong timekeeping?

> **Answer**: (1) Check-in: Lock expire → check-in bi duplicate → DB UNIQUE constraint bat → insert bi fail → checkin API tra loi error thay vi 200 → UX xau nhung data OK. (2) Session: User dang request co the gap error → retry tu dong hoac phai login lai. (3) Cache: Miss → fallback toi DB → DB tai tang tam thoi → cần DB co du capacity. (4) Rate limit: Counter bi mat → window reset → user co the gui nhieu request hon trong 15 giay do. (5) Stream: XADD fail → attendance event bi mat → worker khong nhan duoc → can monitor va alert. Tong ket: Failover 15 giay la chap nhan duoc voi design dung (DB constraint + retry + fallback). Quan trong la app code phai xu ly Redis connection error gracefully, khong crash.

---

## Appendix: Production Redis Checklist

### Truoc Khi Deploy

- [ ] `maxmemory` dat phu hop (< 60% RAM)
- [ ] `maxmemory-policy` da chon dung
- [ ] `protected-mode yes`
- [ ] `bind` chi tren private interface
- [ ] ACL da cau hinh, default user disabled
- [ ] `FLUSHALL` / `FLUSHDB` da rename hoac disable
- [ ] Persistence phu hop cho use case
- [ ] `slowlog-log-slower-than` da bat
- [ ] `latency-monitor-threshold` da bat

### Monitoring

- [ ] Prometheus Redis Exporter da chay
- [ ] Alert tren memory (70%, 85%)
- [ ] Alert tren eviction
- [ ] Alert tren replication lag
- [ ] Alert tren hit ratio (< 80%)
- [ ] Alert tren rejected_connections
- [ ] SLOWLOG review hang tuan

### Ha Tang

- [ ] Redis khong tren cung server voi DB
- [ ] Network latency Redis < 1ms
- [ ] SSD cho persistence
- [ ] Sentinel hoac Cluster cho production
- [ ] Backup RDB dinh ky (du co AOF)
- [ ] Runbook cho failover

### Application

- [ ] Redis client dung singleton pattern
- [ ] Retry logic cho Redis error
- [ ] Fallback khi Redis down (graceful degradation)
- [ ] Khong luu sensitive data khong ma hoa
- [ ] TTL co tren moi cache key

---

*Day 4 hoan thanh. Redis Production series ket thuc.*
*Next: Ap dung vao Timekeeping System — implement tung layer.*
