# Day 3: Redis Patterns — Production Guide

> **Mindset**: Day 3 la cau noi giua "biet Redis" va "dung Redis dung cach trong production". Pattern o day khong phai concept — la code thuc te, edge case thuc te, incident thuc te.

---

## Muc Tieu

Sau Day 3, ban phai:

- [ ] Implement duoc Cache-Aside pattern voi TTL jitter
- [ ] Biet 3 cache failure modes va cach xu ly tung mode
- [ ] Implement duoc distributed lock dung cach (acquire, hold, release)
- [ ] Implement duoc idempotency key cho retry-safe API
- [ ] Implement duoc rate limiter (fixed window va sliding window)
- [ ] Thiet ke duoc full queue/retry/DLQ flow voi Streams
- [ ] Hieu tai sao Redis khong thay the DB constraint

**Context Project**: Timekeeping System. Check-in API chiu tai cua 2000 employee vao luc 8h sang.

---

## Keywords Day 3

```
Redis cache aside pattern
Redis cache invalidation (delete vs update)
Redis TTL jitter cache stampede prevention
Redis cache stampede thundering herd
Redis cache penetration null caching
Redis cache avalanche TTL spread
Redis hot key sharding local cache
Redis distributed lock SET NX PX token
Redis lock renewal watchdog
Redis idempotency key NX EX
Redis rate limiting fixed window sliding window token bucket leaky bucket
Redis Streams retry dead letter queue XPENDING XCLAIM
Redis singleflight pattern
Redis stale-while-revalidate
Redis cache warming
```

---

## 1. Cache-Aside Pattern

### 1.1 Read Flow

```
Client request employee 100
          ↓
GET cache:employee:100
          ↓
    Cache HIT?
   /          \
  YES          NO
   |            |
Return       Query PostgreSQL
JSON         employees WHERE id=100
              |
             SET cache:employee:100
               JSON.stringify(employee)
               EX (600 + random 0-60)
              |
            Return
```

```typescript
// NestJS - Employee Service
async findById(id: number): Promise<Employee | null> {
  const cacheKey = `cache:employee:${id}`;

  // 1. Try cache
  const cached = await this.redis.get(cacheKey);
  if (cached !== null) {
    return JSON.parse(cached);
  }

  // 2. Cache miss: query DB
  const employee = await this.employeeRepo.findOne({ where: { id } });

  // 3. Set cache (kể cả null — xem Cache Penetration)
  if (employee) {
    const ttl = 600 + Math.floor(Math.random() * 60); // jitter
    await this.redis.set(cacheKey, JSON.stringify(employee), 'EX', ttl);
  } else {
    // Null caching: tranh cache penetration
    await this.redis.set(cacheKey, 'NULL', 'EX', 30);
  }

  return employee;
}
```

### 1.2 Write / Update Flow

**Rule quan trong**: Khi update data, **DEL cache** (khong update cache).

```
Update employee 100
     ↓
UPDATE PostgreSQL employees SET ... WHERE id=100
     ↓
DEL cache:employee:100      ← Khong SET gia tri moi
     ↓
Request tiep theo se miss cache → query DB → set cache moi
```

**Tai sao DEL chu khong SET?**

| Strategy | Van de |
|---|---|
| SET cache sau DB write | Race condition: 2 concurrent writes, write nao set cache sau cung thang — co the la stale value |
| DEL cache sau DB write | An toan hon — request tiep theo luon lay fresh data tu DB |

```typescript
async updateEmployee(id: number, dto: UpdateEmployeeDto): Promise<Employee> {
  // 1. Update DB (source of truth)
  const employee = await this.employeeRepo.update(id, dto);

  // 2. Invalidate cache — khong set gia tri moi
  await this.redis.unlink(`cache:employee:${id}`);

  // 3. Neu co cac cache phu thuoc, invalidate het
  await this.redis.unlink(`cache:attendance-summary:${id}:*`); // Lua hoac pipeline
  // Hoac dung tag-based invalidation (nang cao)

  return employee;
}
```

### 1.3 Cache Invalidation La Bai Toan Kho

> "There are only two hard things in Computer Science: cache invalidation and naming things." — Phil Karlton

**Van de thuc te**: Employee thay doi department. Cache cua:
- `cache:employee:100` → Phai del
- `cache:department:10:employees` → Phai del
- `cache:attendance-summary:100:*` → Nen del

**Giai phap pragmatic cho timekeeping**:
1. TTL ngan + accept eventual consistency — don gian nhat
2. Event-driven invalidation: update DB → PUBLISH event → subscriber del cac cache lien quan
3. Tag-based invalidation (phuc tap): Moi cache key duoc tag, del theo tag

**Rule production**: Dung TTL ngan hon la them logic invalidation phuc tap. Dung invalidation logic chi khi consistency requirement ro rang.

---

## 2. TTL Strategy

### 2.1 Bang TTL Timekeeping System

| Cache Key | TTL | Ly Do | Stale OK? |
|---|---|---|---|
| `cache:employee:{id}` | 600 + jitter(0-60)s | Profile it thay doi | Yes, 10 phut |
| `cache:department:{id}` | 1800 + jitter(0-120)s | Department it thay doi | Yes, 30 phut |
| `cache:shift:{id}` | 3600 + jitter(0-300)s | Shift rat it thay doi | Yes, 1 gio |
| `cache:attendance-summary:{id}:{date}` | 60 + jitter(0-30)s | Update lien tuc | Yes, 1-2 phut |
| `cache:work-schedule:{id}:{week}` | 900 + jitter(0-60)s | Thay doi it | Yes, 15 phut |
| `session:{token}` | 28800 | 8 gio session | No |
| `otp:{phone}` | 180 | 3 phut OTP | No |
| `otp:attempt:{phone}` | 900 | 15 phut rate limit | No |
| `rate:login:{id}` | 900 | 15 phut window | No |
| `rate:checkin:{id}` | 60 | 1 phut window | No |
| `lock:checkin:{id}:{date}` | 10 | Short critical section | No |
| `idempotency:checkin:{req_id}` | 86400 | 24 gio de-dup | No |
| `idempotency:export:{req_id}` | 3600 | 1 gio export de-dup | No |

### 2.2 TTL Jitter

```typescript
// Khong jitter - nguy hiem
const TTL = 600;
await redis.set(key, value, 'EX', TTL);
// Neu 2000 employee cache duoc set luc 8:00:00,
// tat ca expire luc 8:10:00 → thundering herd

// Co jitter - an toan
const BASE_TTL = 600;
const JITTER = Math.floor(Math.random() * 60); // 0-59 giay
await redis.set(key, value, 'EX', BASE_TTL + JITTER);
// Expire trai ra tu 8:10:00 den 8:10:59 → giam tai
```

### 2.3 Stale-While-Revalidate Pattern

Tra ve data cu trong khi refresh background — zero latency cho client.

```typescript
async getWithSWR(key: string, fetcher: () => Promise<any>, ttl: number) {
  const raw = await this.redis.get(key);

  if (raw) {
    const { value, expiresAt } = JSON.parse(raw);
    const remainingTtl = expiresAt - Date.now();

    // Neu con > 20% TTL, tra ve ngay
    if (remainingTtl > ttl * 1000 * 0.2) {
      return value;
    }

    // Con < 20% TTL: tra ve stale, refresh background
    setImmediate(async () => {
      const fresh = await fetcher();
      const payload = JSON.stringify({
        value: fresh,
        expiresAt: Date.now() + ttl * 1000
      });
      await this.redis.set(key, payload, 'EX', ttl + 60);
    });

    return value; // Tra stale, client khong doi
  }

  // Cache miss: fetch va cache
  const fresh = await fetcher();
  const payload = JSON.stringify({
    value: fresh,
    expiresAt: Date.now() + ttl * 1000
  });
  await this.redis.set(key, payload, 'EX', ttl + 60);
  return fresh;
}
```

---

## 3. Cache Failure Modes

### 3.1 Cache Stampede (Thundering Herd)

**Mo ta**: Mot cache key popular expire → hang tram request cung luc miss cache → tat ca query DB → DB spike → co the down.

**Boi canh timekeeping**: 8:00 AM, 2000 employee check-in, cache attendance-summary expire, 2000 request cung hit DB.

**Giai phap 1: TTL Jitter** (xem 2.2) — Don gian nhat.

**Giai phap 2: Mutex per Key (Cache Lock)**

```typescript
async getWithMutex(key: string, fetcher: () => Promise<any>, ttl: number) {
  // Thu lay tu cache
  const cached = await this.redis.get(key);
  if (cached) return JSON.parse(cached);

  // Cache miss: Acquire mutex
  const lockKey = `lock:cache-rebuild:${key}`;
  const lockToken = uuidv4();
  const acquired = await this.redis.set(lockKey, lockToken, 'NX', 'EX', 10);

  if (acquired) {
    // Process nay thang lock: fetch va set cache
    try {
      const data = await fetcher();
      const ttlWithJitter = ttl + Math.floor(Math.random() * 60);
      await this.redis.set(key, JSON.stringify(data), 'EX', ttlWithJitter);
      return data;
    } finally {
      // Release lock
      await this.releaseLock(lockKey, lockToken);
    }
  } else {
    // Process khac dang rebuild: doi va retry
    await new Promise(resolve => setTimeout(resolve, 50));
    const retried = await this.redis.get(key);
    return retried ? JSON.parse(retried) : await fetcher();
  }
}
```

**Giai phap 3: Singleflight Pattern**

```typescript
// Chi goi fetcher 1 lan du co 100 concurrent request
import { SingleFlight } from 'async-singleflight';

private readonly singleFlight = new SingleFlight();

async get(key: string, fetcher: () => Promise<any>, ttl: number) {
  const cached = await this.redis.get(key);
  if (cached) return JSON.parse(cached);

  // Tat ca concurrent request cho cung key chi goi fetcher 1 lan
  return this.singleFlight.do(key, async () => {
    const data = await fetcher();
    await this.redis.set(key, JSON.stringify(data), 'EX', ttl);
    return data;
  });
}
```

---

### 3.2 Cache Penetration

**Mo ta**: Request query data khong ton tai trong DB (employee ID = 999999) → moi request deu miss cache → hit DB → DB scan vo ich.

**Boi canh**: Attacker gui hang nghin request voi ID khong ton tai.

**Giai phap 1: Null Caching**

```typescript
const employee = await this.employeeRepo.findOne({ where: { id } });

if (employee) {
  await this.redis.set(cacheKey, JSON.stringify(employee), 'EX', 600);
} else {
  // Cache ket qua null voi TTL ngan
  await this.redis.set(cacheKey, '__NULL__', 'EX', 30); // 30 giay
}

// Khi doc:
const raw = await this.redis.get(cacheKey);
if (raw === '__NULL__') return null; // Biet la null, khong query DB
if (raw) return JSON.parse(raw);
```

**Giai phap 2: Bloom Filter**

```
Khi start: Load tat ca valid employee IDs vao Bloom Filter
Request employee 999999 → Bloom Filter tra ve: "Chac chan khong ton tai"
→ Tra ve 404 ngay, khong query cache, khong query DB
```

Redis co RedisBloom module ho tro native Bloom Filter. Tuy nhien, co the dung `redis-bloom` hoac implement o application layer voi `bloom-filters` npm package.

**Giai phap 3: Input Validation** — Don gian nhat cho timekeeping:
- Employee ID phai la so nguyen, trong range hop le
- Validate truoc khi hit cache/DB

---

### 3.3 Cache Avalanche

**Mo ta**: Nhieu cache key expire cung luc → tat ca request hit DB cung luc → DB qua tai.

**Khac stampede**: Stampede = 1 key popular. Avalanche = nhieu key cung expire.

**Boi canh**: Morning cache warming set tat ca 2000 employee cache voi cung TTL = 600s → luc 8:10 tat ca expire cung luc.

**Giai phap**:

```typescript
// 1. TTL Jitter khi set cache
const ttl = baseTtl + Math.floor(Math.random() * maxJitter);

// 2. Staggered cache warming
async warmCache(employees: Employee[]): Promise<void> {
  const pipeline = this.redis.pipeline();
  employees.forEach((emp, index) => {
    // Stagger TTL: moi employee them 1 giay
    const ttl = 600 + (index % 300); // Spread qua 5 phut
    pipeline.set(`cache:employee:${emp.id}`, JSON.stringify(emp), 'EX', ttl);
  });
  await pipeline.exec();
}

// 3. Cache warming truoc khi expire
// Monitor TTL, warm lai khi con 20%
```

---

### 3.4 Hot Key

**Mo ta**: Mot key bi request qua nhieu (vd: "employee:1" cua CEO, "config:global"). Redis single-threaded → hot key tro thanh bottleneck.

**Boi canh**: 2000 employee xem lich lam viec chung → tat ca doc cung 1 `cache:work-schedule:global`.

**Giai phap 1: Local In-Memory Cache (L1 Cache)**

```typescript
import NodeCache from 'node-cache';

private localCache = new NodeCache({ stdTTL: 10 }); // 10 giay

async getHotKey(key: string): Promise<any> {
  // L1: In-memory cache (process local)
  const local = this.localCache.get(key);
  if (local !== undefined) return local;

  // L2: Redis
  const redis = await this.redis.get(key);
  if (redis) {
    const value = JSON.parse(redis);
    this.localCache.set(key, value); // Cache o local
    return value;
  }

  // L3: DB
  const data = await this.fetchFromDB(key);
  await this.redis.set(key, JSON.stringify(data), 'EX', 600);
  this.localCache.set(key, data);
  return data;
}
```

**Giai phap 2: Key Sharding**

```typescript
// Thay vi 1 hot key, dung N keys (shard theo request)
const SHARD_COUNT = 10;
const shardId = Math.floor(Math.random() * SHARD_COUNT);
const key = `cache:work-schedule:global:shard:${shardId}`;

// Moi request doc tu 1 shard ngau nhien
// Load trai deu tren 10 key thay vi 1
```

---

## 4. Distributed Lock

### 4.1 Acquire Lock

```typescript
import { v4 as uuidv4 } from 'uuid';

async acquireLock(
  resource: string,
  ttlMs: number = 10000
): Promise<string | null> {
  const lockKey = `lock:${resource}`;
  const token = uuidv4();

  // SET NX PX = atomic "set if not exists" voi TTL milliseconds
  const result = await this.redis.set(lockKey, token, 'NX', 'PX', ttlMs);

  return result === 'OK' ? token : null;
}
```

### 4.2 Release Lock (Phai Dung Lua)

```typescript
private readonly RELEASE_LOCK_SCRIPT = `
  if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
  else
    return 0
  end
`;

async releaseLock(resource: string, token: string): Promise<boolean> {
  const lockKey = `lock:${resource}`;
  const result = await this.redis.eval(
    this.RELEASE_LOCK_SCRIPT,
    1,
    lockKey,
    token
  );
  return result === 1;
}
```

**Tai sao phai Lua?** DEL khong kiem tra value → co the xoa lock cua process khac:

```
T1: Process A acquire lock, token = "abc", TTL = 10s
T2: Process A bi timeout (GC pause, slow query)
T3: Lock expire (het 10s)
T4: Process B acquire lock, token = "xyz"
T5: Process A resume, goi DEL lock:checkin:100:2026-07-08
    → Xoa lock cua Process B!
T6: Process C cung acquire lock → 2 process cung run critical section
```

### 4.3 Lock Renewal (Watchdog cho Long-running Job)

```typescript
async runWithLock<T>(
  resource: string,
  job: () => Promise<T>,
  lockTtlMs = 10000
): Promise<T> {
  const token = await this.acquireLock(resource, lockTtlMs);
  if (!token) {
    throw new Error(`Could not acquire lock for ${resource}`);
  }

  const lockKey = `lock:${resource}`;
  const renewInterval = Math.floor(lockTtlMs / 3); // Renew sau 1/3 TTL

  // Watchdog: extend TTL trong khi job chay
  const watchdog = setInterval(async () => {
    const current = await this.redis.get(lockKey);
    if (current === token) {
      await this.redis.pexpire(lockKey, lockTtlMs);
    }
  }, renewInterval);

  try {
    return await job();
  } finally {
    clearInterval(watchdog);
    await this.releaseLock(resource, token);
  }
}

// Usage:
await this.lockService.runWithLock(
  `checkin:${empId}:${date}`,
  async () => {
    // Critical section: insert attendance record
    await this.attendanceRepo.insert({ employee_id: empId, work_date: date });
  },
  10000 // 10 giay lock TTL
);
```

### 4.4 Rules Distributed Lock

| Rule | Ly Do |
|---|---|
| Luon co TTL (PX/EX) | Tranh lock mai mai neu process crash |
| Luon co unique token | Tranh xoa lock cua process khac |
| Release bang Lua | Atomic check-and-delete |
| TTL ngan (5-30s) | Tranh lock bi giu qua lau |
| Watchdog neu job dai hon TTL | Tranh lock expire giua chung |
| DB constraint la chot cuoi | Lock Redis co the fail → DB UNIQUE constraint bat loi |

```sql
-- PostgreSQL: DB la chot cuoi cung
CREATE TABLE attendances (
  employee_id BIGINT NOT NULL,
  work_date DATE NOT NULL,
  check_in_time TIMESTAMP,
  CONSTRAINT uq_attendance_emp_date UNIQUE (employee_id, work_date)
);
```

**Redis lock tranh duplicate call. DB UNIQUE constraint tranh duplicate data.** Ca hai can co mat.

---

## 5. Idempotency Key

### 5.1 Van De Idempotency

```
Client gui POST /checkin
Server xu ly: 200ms
Network timeout o client (300ms) → Client khong nhan response
Client retry: gui lai POST /checkin
Server xu ly lan 2: → Duplicate check-in!
```

**Giai phap**: Idempotency key — moi request co unique request_id, server de-dup.

### 5.2 Idempotency Flow

```
Client gui:
  POST /checkin
  Headers: X-Idempotency-Key: req-550e8400-e29b-41d4-a716-446655440000
  Body: { employee_id: 100, timestamp: "2026-07-08T08:00:00" }

Server:
  1. key = idempotency:checkin:req-550e8400-...
  2. SET key "processing" NX EX 86400
     - "OK" → Request moi, xu ly tiep
     - nil → Key ton tai, la duplicate

  3. Neu la duplicate:
     - GET key
     - Neu "processing" → Tra ve 202 Accepted (dang xu ly)
     - Neu JSON result → Tra ve ket qua cu (idempotent response)

  4. Xu ly: insert PostgreSQL
  5. SET key JSON.stringify(result) EX 86400  (luu ket qua)

Client nhan ket qua, du retry nhieu lan, response giong nhau.
```

```typescript
// NestJS Interceptor for Idempotency
@Injectable()
export class IdempotencyInterceptor implements NestInterceptor {
  constructor(private redis: Redis) {}

  async intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const req = context.switchToHttp().getRequest();
    const idempotencyKey = req.headers['x-idempotency-key'];

    if (!idempotencyKey) {
      return next.handle(); // Khong co key → pass through
    }

    const redisKey = `idempotency:${req.path}:${idempotencyKey}`;

    // Try set "processing"
    const acquired = await this.redis.set(redisKey, 'processing', 'NX', 'EX', 86400);

    if (!acquired) {
      // Duplicate request
      const stored = await this.redis.get(redisKey);
      if (stored === 'processing') {
        throw new HttpException('Request is being processed', 202);
      }
      // Tra ve ket qua cu
      return of(JSON.parse(stored));
    }

    // Xu ly request moi
    return next.handle().pipe(
      tap(async (response) => {
        // Luu ket qua de tra cho retry sau
        await this.redis.set(redisKey, JSON.stringify(response), 'EX', 86400);
      }),
      catchError(async (error) => {
        // Xoa key neu loi de client co the retry
        await this.redis.unlink(redisKey);
        throw error;
      })
    );
  }
}
```

---

## 6. Rate Limiting

### 6.1 Fixed Window Counter

```typescript
async checkRateLimit(
  key: string,
  limit: number,
  windowSeconds: number
): Promise<{ allowed: boolean; remaining: number; resetAt: number }> {
  const redisKey = `rate:${key}`;

  // Lua script atomic: INCR + set TTL neu la request dau tien
  const script = `
    local current = redis.call('GET', KEYS[1])
    if current == false then
      redis.call('SET', KEYS[1], 1, 'EX', ARGV[2])
      return {1, tonumber(ARGV[1]) - 1}
    end
    local new_val = redis.call('INCR', KEYS[1])
    return {new_val, math.max(0, tonumber(ARGV[1]) - new_val)}
  `;

  const [count, remaining] = await this.redis.eval(
    script, 1, redisKey, limit, windowSeconds
  ) as [number, number];

  const ttl = await this.redis.ttl(redisKey);

  return {
    allowed: count <= limit,
    remaining: Math.max(0, remaining),
    resetAt: Date.now() + ttl * 1000
  };
}
```

**Diem yeu Fixed Window**: Boundary issue.

```
Window: 0-60 giay. Limit: 10 req/phut
T=59.0s: 10 requests → da day window
T=60.0s: Window reset
T=60.5s: 10 requests moi → OK theo window

Thuc te: 20 requests trong 1.5 giay (59.0 den 60.5)
```

### 6.2 Sliding Window (Chinh Xac Hon)

```typescript
async checkSlidingRateLimit(
  key: string,
  limit: number,
  windowMs: number
): Promise<{ allowed: boolean; remaining: number }> {
  const redisKey = `rate:sliding:${key}`;
  const now = Date.now();
  const windowStart = now - windowMs;
  const requestId = `${now}-${Math.random()}`;

  const script = `
    local key = KEYS[1]
    local window_start = ARGV[1]
    local now = ARGV[2]
    local request_id = ARGV[3]
    local limit = tonumber(ARGV[4])
    local window_ms = tonumber(ARGV[5])

    -- Xoa request cu ngoai window
    redis.call('ZREMRANGEBYSCORE', key, '-inf', window_start)

    -- Dem request trong window
    local count = redis.call('ZCARD', key)

    if count < limit then
      -- Cho phep: them request moi
      redis.call('ZADD', key, now, request_id)
      redis.call('PEXPIRE', key, window_ms * 2)
      return {1, limit - count - 1}
    else
      return {0, 0}
    end
  `;

  const [allowed, remaining] = await this.redis.eval(
    script, 1, redisKey,
    windowStart, now, requestId, limit, windowMs
  ) as [number, number];

  return { allowed: allowed === 1, remaining };
}
```

### 6.3 Rate Limit cho Cac Endpoint Timekeeping

```typescript
// Cau hinh rate limit
const RATE_LIMITS = {
  'checkin': { limit: 3, windowSeconds: 60 },     // 3 check-in/phut
  'login': { limit: 5, windowSeconds: 900 },       // 5 lan login fail/15 phut
  'otp-send': { limit: 3, windowSeconds: 300 },    // 3 OTP/5 phut
  'export-report': { limit: 2, windowSeconds: 60 }, // 2 export/phut
  'api-global': { limit: 100, windowSeconds: 60 }, // 100 req/phut moi employee
};

// Guard/Middleware
@Injectable()
export class RateLimitGuard implements CanActivate {
  async canActivate(context: ExecutionContext): Promise<boolean> {
    const req = context.switchToHttp().getRequest();
    const endpoint = this.getEndpointKey(req);
    const employeeId = req.user?.id;
    const config = RATE_LIMITS[endpoint];

    if (!config || !employeeId) return true;

    const { allowed, remaining } = await this.rateLimiter.checkRateLimit(
      `${endpoint}:${employeeId}`,
      config.limit,
      config.windowSeconds
    );

    if (!allowed) {
      throw new HttpException({
        statusCode: 429,
        message: 'Too Many Requests',
        remaining: 0,
      }, 429);
    }

    return true;
  }
}
```

### 6.4 Rate Limit Headers (Best Practice)

```typescript
// Tra ve headers cho client biet trang thai
res.setHeader('X-RateLimit-Limit', limit);
res.setHeader('X-RateLimit-Remaining', remaining);
res.setHeader('X-RateLimit-Reset', resetAt);
res.setHeader('Retry-After', Math.ceil((resetAt - Date.now()) / 1000));
```

---

## 7. Queue / Retry / DLQ

### 7.1 Architecture

```
Check-in API
     |
XADD stream:attendance-events * emp_id 100 type check_in

     |
     ├─── Consumer Group: notification-workers
     |         XREADGROUP → Send notification → XACK
     |
     ├─── Consumer Group: db-sync-workers
     |         XREADGROUP → Write PostgreSQL summary → XACK
     |
     └─── Consumer Group: audit-workers
               XREADGROUP → Write audit log → XACK
```

### 7.2 Producer

```typescript
// CheckIn Service
async processCheckIn(dto: CheckInDto): Promise<void> {
  // ... business logic ...

  // Publish event to stream
  await this.redis.xadd(
    'stream:attendance-events',
    'MAXLEN', '~', '50000',  // Giu toi da ~50k messages
    '*',                      // Auto-generate ID
    'emp_id', dto.employeeId.toString(),
    'type', 'check_in',
    'timestamp', new Date().toISOString(),
    'work_date', dto.workDate,
    'location_lat', dto.lat?.toString() ?? '',
    'location_lng', dto.lng?.toString() ?? ''
  );
}
```

### 7.3 Consumer voi Retry Logic

```typescript
@Injectable()
export class AttendanceStreamConsumer implements OnModuleInit {
  private readonly STREAM_KEY = 'stream:attendance-events';
  private readonly GROUP_NAME = 'notification-workers';
  private readonly CONSUMER_NAME = `worker-${process.pid}`;
  private readonly MAX_RETRIES = 3;
  private readonly PENDING_TIMEOUT_MS = 30000; // 30 giay

  async onModuleInit() {
    // Tao consumer group neu chua co
    try {
      await this.redis.xgroup(
        'CREATE', this.STREAM_KEY, this.GROUP_NAME, '$', 'MKSTREAM'
      );
    } catch (e) {
      if (!e.message.includes('BUSYGROUP')) throw e;
    }

    // Start consuming
    this.consume();
    this.processPending(); // Xu ly message pending (bi drop)
  }

  private async consume() {
    while (true) {
      try {
        const messages = await this.redis.xreadgroup(
          'GROUP', this.GROUP_NAME, this.CONSUMER_NAME,
          'COUNT', '10',
          'BLOCK', '5000', // Wait 5 giay
          'STREAMS', this.STREAM_KEY, '>'
        );

        if (!messages) continue;

        for (const [, entries] of messages) {
          for (const [id, fields] of entries) {
            await this.handleMessage(id, this.parseFields(fields));
          }
        }
      } catch (error) {
        this.logger.error('Stream consumer error', error);
        await new Promise(r => setTimeout(r, 1000)); // Tranh tight loop
      }
    }
  }

  private async handleMessage(id: string, data: Record<string, string>) {
    try {
      await this.sendNotification(data);

      // Xu ly thanh cong → ACK
      await this.redis.xack(this.STREAM_KEY, this.GROUP_NAME, id);
      this.logger.log(`ACK message ${id}`);

    } catch (error) {
      this.logger.warn(`Failed to process ${id}`, error);
      // Khong ACK → message o trong pending list
      // Sau PENDING_TIMEOUT_MS, processPending() se claim va retry
    }
  }

  // Xu ly cac message pending (chua duoc ACK sau timeout)
  private async processPending() {
    while (true) {
      await new Promise(r => setTimeout(r, 10000)); // Chay moi 10 giay

      try {
        // Lay message da pending > 30s
        const pending = await this.redis.xautoclaim(
          this.STREAM_KEY,
          this.GROUP_NAME,
          this.CONSUMER_NAME,
          this.PENDING_TIMEOUT_MS,
          '0-0',
          'COUNT', '20'
        );

        const [, messages] = pending;
        if (!messages?.length) continue;

        for (const [id, fields] of messages) {
          const data = this.parseFields(fields);
          const deliveryCount = await this.getDeliveryCount(id);

          if (deliveryCount > this.MAX_RETRIES) {
            // Vuot qua so lan retry → chuyen sang DLQ
            await this.moveToDLQ(id, data, deliveryCount);
            await this.redis.xack(this.STREAM_KEY, this.GROUP_NAME, id);
          } else {
            // Retry
            await this.handleMessage(id, data);
          }
        }
      } catch (error) {
        this.logger.error('Pending processor error', error);
      }
    }
  }

  private async moveToDLQ(
    originalId: string,
    data: Record<string, string>,
    retries: number
  ) {
    this.logger.error(`Moving ${originalId} to DLQ after ${retries} retries`, data);

    await this.redis.xadd(
      'stream:attendance-events:dlq',
      'MAXLEN', '~', '10000',
      '*',
      'original_id', originalId,
      'data', JSON.stringify(data),
      'retries', retries.toString(),
      'failed_at', new Date().toISOString(),
      'reason', 'max_retries_exceeded'
    );

    // Alert: Slack, PagerDuty, email...
    await this.alertService.send(`DLQ: attendance event ${originalId} failed after ${retries} retries`);
  }

  private async getDeliveryCount(messageId: string): Promise<number> {
    const pending = await this.redis.xpending(
      this.STREAM_KEY, this.GROUP_NAME,
      '-', '+', 1,
      this.CONSUMER_NAME
    );
    // Tim message cu the trong pending
    const entry = pending.find(([id]) => id === messageId);
    return entry ? entry[3] : 0; // delivery-count
  }

  private parseFields(fields: string[]): Record<string, string> {
    const result: Record<string, string> = {};
    for (let i = 0; i < fields.length; i += 2) {
      result[fields[i]] = fields[i + 1];
    }
    return result;
  }
}
```

### 7.4 DLQ Monitor

```typescript
// Monitor DLQ dinh ky
@Cron('*/5 * * * *') // Moi 5 phut
async monitorDLQ() {
  const dlqLength = await this.redis.xlen('stream:attendance-events:dlq');
  if (dlqLength > 0) {
    this.logger.error(`DLQ has ${dlqLength} unprocessed messages`);
    // Send alert
  }
}

// Manual reprocess DLQ
async reprocessDLQ() {
  const messages = await this.redis.xrange('stream:attendance-events:dlq', '-', '+', 'COUNT', 100);
  for (const [id, fields] of messages) {
    const data = JSON.parse(fields.find((_, i) => fields[i-1] === 'data') || '{}');
    // Reprocess manually hoac push lai vao main stream
    await this.redis.xadd('stream:attendance-events', 'MAXLEN', '~', '50000', '*', ...Object.entries(data).flat());
    await this.redis.xdel('stream:attendance-events:dlq', id);
  }
}
```

---

## 8. Full Check-in Flow — Bai Tap Lon

### 8.1 Architecture tong the

```
POST /checkin
  Body: { employee_id: 100, timestamp: "2026-07-08T08:00:00" }
  Headers: X-Idempotency-Key: req-uuid-abc

         ↓
  [1] Rate Limit Guard
      rate:checkin:100 → INCR → limit 3/phut
      Neu vuot → 429 Too Many Requests

         ↓
  [2] Idempotency Check
      idempotency:checkin:req-uuid-abc
      SET NX → "processing"
      Neu ton tai → tra ket qua cu hoac 202

         ↓
  [3] Acquire Distributed Lock
      lock:checkin:100:2026-07-08 → SET NX PX 10000 token123
      Neu fail → 409 Conflict (dang xu ly)

         ↓
  [4] Write PostgreSQL
      INSERT INTO attendances (employee_id, work_date, check_in_time)
      ON CONFLICT (employee_id, work_date) DO NOTHING
      → DB UNIQUE constraint la chot cuoi

         ↓
  [5] Invalidate Cache
      UNLINK cache:attendance-summary:100:2026-07-08

         ↓
  [6] Publish to Stream
      XADD stream:attendance-events * emp_id 100 type check_in ...

         ↓
  [7] Release Lock
      Lua: check token → DEL

         ↓
  [8] Set Idempotency Result
      SET idempotency:checkin:req-uuid-abc {"success":true,...} EX 86400

         ↓
  Return 200 { success: true, attendance_id: 123 }

  --- Background (Stream Consumer) ---

  [9] Worker nhan event tu stream
      → Send notification to employee
      → Update realtime dashboard
      → XACK

  [10] Neu worker fail → pending list
       → Sau 30s → XCLAIM → retry
       → Sau 3 lan fail → DLQ
       → Alert
```

### 8.2 Code: CheckIn Service

```typescript
@Injectable()
export class CheckInService {
  async checkIn(
    empId: number,
    workDate: string,
    idempotencyKey: string
  ): Promise<CheckInResult> {

    // [1] Rate Limit
    const { allowed } = await this.rateLimiter.checkRateLimit(
      `checkin:${empId}`, 3, 60
    );
    if (!allowed) throw new TooManyRequestsException();

    // [2] Idempotency
    const idemKey = `idempotency:checkin:${idempotencyKey}`;
    const existing = await this.redis.get(idemKey);
    if (existing === 'processing') throw new ConflictException('Processing');
    if (existing) return JSON.parse(existing); // Idempotent response

    const acquired = await this.redis.set(idemKey, 'processing', 'NX', 'EX', 86400);
    if (!acquired) throw new ConflictException('Duplicate key');

    // [3] Acquire Lock
    const lockResource = `checkin:${empId}:${workDate}`;
    const lockToken = await this.lockService.acquireLock(lockResource, 10000);
    if (!lockToken) throw new ConflictException('Check-in already in progress');

    try {
      // [4] Write PostgreSQL
      const attendance = await this.attendanceRepo
        .createQueryBuilder()
        .insert()
        .into(Attendance)
        .values({ employee_id: empId, work_date: workDate, check_in_time: new Date() })
        .onConflict('(employee_id, work_date) DO NOTHING')
        .returning('*')
        .execute();

      const result = { success: true, attendanceId: attendance.raw[0]?.id };

      // [5] Invalidate Cache
      await this.redis.unlink(`cache:attendance-summary:${empId}:${workDate}`);

      // [6] Publish to Stream
      await this.redis.xadd(
        'stream:attendance-events', 'MAXLEN', '~', '50000', '*',
        'emp_id', empId.toString(),
        'type', 'check_in',
        'work_date', workDate,
        'attendance_id', result.attendanceId?.toString() ?? ''
      );

      // [8] Save idempotency result
      await this.redis.set(idemKey, JSON.stringify(result), 'EX', 86400);

      return result;

    } catch (error) {
      // Xoa idem key de client co the retry
      await this.redis.unlink(idemKey);
      throw error;
    } finally {
      // [7] Release Lock
      await this.lockService.releaseLock(lockResource, lockToken);
    }
  }
}
```

---

## 9. Checklist Day 3

### Kien thuc

- [ ] Giai thich Cache-Aside pattern: read flow, write/invalidation flow
- [ ] Giai thich tai sao DEL cache sau update, khong SET gia tri moi
- [ ] Neu 3 cache failure modes: stampede, penetration, avalanche
- [ ] Giai thich Singleflight va Mutex-per-key giai quyet stampede
- [ ] Giai thich Null Caching giai quyet penetration
- [ ] Giai thich Distributed Lock: acquire, hold, release dung cach
- [ ] Giai thich tai sao DB UNIQUE constraint van can ben canh Redis lock
- [ ] Giai thich Idempotency Key flow cho retry-safe API
- [ ] So sanh Fixed Window vs Sliding Window rate limit
- [ ] Mo ta Queue/Retry/DLQ flow voi Streams

### Thuc hanh

- [ ] Implement Cache-Aside voi TTL jitter va null caching
- [ ] Implement distributed lock acquire/release (voi Lua)
- [ ] Implement rate limiter (fixed window, Lua atomic)
- [ ] Implement idempotency key interceptor
- [ ] Implement stream producer (XADD)
- [ ] Implement stream consumer voi retry (XREADGROUP + XACK)
- [ ] Implement pending processor (XAUTOCLAIM)
- [ ] Implement DLQ move logic

---

## 10. Production Pitfalls Day 3

### P1: Invalidation Race Condition

```
T1: Request A: GET employee:100 → cache miss → query DB
T2: Request B: UPDATE employee:100 + DEL cache:employee:100
T3: Request A: SET cache:employee:100 = OLD DATA (data truoc update!)
```

**Hau qua**: Cache bi poison voi data cu.

**Giai phap**:
- Chap nhan eventual consistency voi TTL ngan (most cases)
- Dung versioning: cache key bao gom version number → update data → increment version → cache cu tu bo qua

---

### P2: Lock TTL Qua Ngan

```
Job can 15 giay nhung lock TTL = 10 giay
T=10s: Lock expire
T=11s: Process B acquire lock cung resource
T=12s: Process A va B cung run critical section
```

**Giai phap**: Watchdog / lock renewal. Hoac tinh TTL dua tren expected job duration x 2.

---

### P3: Idempotency Key Xung Dot

```
Client A: X-Idempotency-Key: req-123
Client B: X-Idempotency-Key: req-123  (trung lap — bug phia client)
```

**Giai phap**: Idempotency key phai unique per user. Key = `idempotency:{user_id}:{request_uuid}` thay vi chi `{request_uuid}`.

---

### P4: Rate Limit Bypass voi Distributed System

```
Load balancer → Server 1: rate:login:100 = 5 (da day)
             → Server 2: rate:login:100 = 0 (server khac, cache khac?)
```

**Giai phap**: Rate limit phai o **centralized Redis**, khong phai local cache. Tat ca server phai dung cung Redis instance/cluster.

---

### P5: Stream Consumer Khong XACK → Pending Phong To

```
Worker xu ly message → App crash truoc khi XACK
→ Message trong pending list
→ Khong co pending processor → Message stuck mai mai
→ Sau nhieu thang: 100k pending entries → memory issue
```

**Giai phap**: Bat buoc phai co pending processor voi XAUTOCLAIM. Monitor XPENDING count. Alert neu > nguong.

---

### P6: Cache Warming Cold Start

```
Redis restart (moi, rong)
→ 2000 employee check-in luc 8h
→ Tat ca 2000 request cache miss
→ 2000 DB query cung luc
→ DB qua tai
```

**Giai phap**: Cache warming sau restart:
```typescript
@OnEvent('app.started')
async warmCacheOnStart() {
  const employees = await this.employeeRepo.find({ where: { status: 'active' } });
  await this.cacheService.warmCache(employees); // Staggered pipeline set
}
```

---

## 11. Senior Review Questions Day 3

**Q1**: Tai sao DEL cache sau DB update, khong SET cache voi gia tri moi?

> **Answer**: Voi concurrent writes, neu 2 process cung update DB sau do SET cache, process nao SET sau se override. Neu update cua process A kết thúc truoc B, nhung SET cache cua A lay sau B (do race condition), cache se luu gia tri cu cua A. DEL an toan hon vi no buoc request tiep theo query fresh data tu DB. Trade-off: DEL → 1 cache miss sau update. SET → risk poison cache.

**Q2**: Mo ta full distributed lock lifecycle trong check-in API.

> **Answer**: (1) SET NX PX 10000 voi unique UUID token. (2) Neu nil → duplicate request dang chay → reject. (3) Chay critical section (INSERT PostgreSQL). (4) Watchdog renew TTL neu job > 10s. (5) Release: Lua check token == current value → DEL. (6) Neu process crash → TTL tu expire, process khac co the acquire sau 10s. (7) DB UNIQUE constraint la safety net cuoi cung.

**Q3**: Cache penetration co the gay DOS khong? Giai quyet the nao?

> **Answer**: Co. Attacker gui 10k request voi employee IDs khong ton tai → 10k DB queries. Giai phap: (1) Input validation — ID phai hop le truoc khi hit cache/DB. (2) Null caching: cache ket qua null voi TTL 30s → request ke tiep hit cache. (3) Bloom Filter — truoc khi query cache, kiem tra ID co kha nang ton tai khong (false negative = 0, false positive nho).

**Q4**: Sliding window rate limit chính xác hon fixed window the nao? Trade-off la gi?

> **Answer**: Fixed window co boundary issue: 3 req cuoi phut 1 + 3 req dau phut 2 = 6 req trong 2 giay. Sliding window dung Sorted Set voi score = timestamp, xoa request ngoai window → luon chinh xac. Trade-off: Sliding window ton O(1) space per request (Sorted Set member), fixed window chi O(1) total per key. Voi limit 10 req/phut, sliding window giu toi da 10 members per key — chap nhan duoc.

**Q5**: Neu Stream consumer bi crash giua luc xu ly 100 message, dieu gi xay ra?

> **Answer**: 100 message o trong pending list, chua duoc XACK. Consumer khac (hoac consumer restart) phai dung XAUTOCLAIM de claim lai cac message da pending > timeout (vd: 30s). Delivery count tang. Neu delivery count > MAX_RETRIES → move sang DLQ. DO vay: (1) Bat buoc co pending processor chay dinh ky. (2) Monitor XPENDING count. (3) Monitor DLQ. (4) Business logic trong consumer phai idempotent (xu ly 2 lan cung OK).

**Q6**: Tai sao noi "Redis lock la tranh, DB constraint la chot"?

> **Answer**: Redis lock giam tai va tranh race condition trong normal case. Nhung Redis co the fail: network partition, Redis restart, lock TTL expire som, bug trong release code. Neu chi dua vao Redis lock, 2 process co the ghi duplicate khi Redis gap van de. DB UNIQUE constraint la atomic, durable, khong co race condition → la lop bao ve cuoi cung. Pattern chuan: Redis lock + DB constraint + idempotent operation.

---

## Appendix: Decision Tree

### Nen Dung Pattern Nao?

```
Can giam DB load cho read-heavy?
  → Cache-Aside

Data hay thay doi va can consistency cao?
  → TTL ngan (30-60s) + accept eventual consistency
  → hoac event-driven invalidation

Nhieu concurrent request cho cung 1 cache key?
  → Mutex per key hoac Singleflight

Query data khong ton tai?
  → Null caching + Input validation

Nhieu cache keys expire cung luc?
  → TTL jitter

Chong duplicate write?
  → Distributed lock (Redis) + DB UNIQUE constraint

API co the bi retry?
  → Idempotency key

Can goi han so request?
  → Fixed window (don gian)
  → Sliding window (chinh xac)

Can xu ly event background?
  → Redis Streams voi consumer group

Event quan trong, khong duoc mat?
  → Streams voi XACK + retry + DLQ
  
Event nhẹ, mat duoc?
  → Pub/Sub (don gian hon)
```

---

*Day 3 hoan thanh. Next: Day 4 — Redis Persistence, Replication & Cluster.*
