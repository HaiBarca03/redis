# Day 3: Redis Patterns — Production Guide

> **Mindset**: Day 3 là cầu nối giữa "biết Redis" và "dùng Redis đúng cách trong production". Pattern ở đây không phải concept — là code thực tế, edge case thực tế, incident thực tế.

---

## Mục Tiêu

Sau Day 3, bạn phải:

- [ ] Implement được Cache-Aside pattern với TTL jitter
- [ ] Biết 3 cache failure modes và cách xử lý từng mode
- [ ] Implement được distributed lock đúng cách (acquire, hold, release)
- [ ] Implement được idempotency key cho retry-safe API
- [ ] Implement được rate limiter (fixed window và sliding window)
- [ ] Thiết kế được full queue/retry/DLQ flow với Streams
- [ ] Hiểu tại sao Redis không thay thế DB constraint

**Context Project**: Timekeeping System. Check-in API chịu tải của 2000 employee vào lúc 8h sáng.

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
    // Null caching: tránh cache penetration
    await this.redis.set(cacheKey, 'NULL', 'EX', 30);
  }

  return employee;
}
```

### 1.2 Write / Update Flow

**Rule quan trọng**: Khi update data, **DEL cache** (không update cache).

```
Update employee 100
     ↓
UPDATE PostgreSQL employees SET ... WHERE id=100
     ↓
DEL cache:employee:100      ← Không SET giá trị mới
     ↓
Request tiếp theo sẽ miss cache → query DB → set cache mới
```

**Tại sao DEL chứ không SET?**

| Strategy | Vấn đề |
|---|---|
| SET cache sau DB write | Race condition: 2 concurrent writes, write nào set cache sau cùng thắng — có thể là stale value |
| DEL cache sau DB write | An toàn hơn — request tiếp theo luôn lấy fresh data từ DB |

```typescript
async updateEmployee(id: number, dto: UpdateEmployeeDto): Promise<Employee> {
  // 1. Update DB (source of truth)
  const employee = await this.employeeRepo.update(id, dto);

  // 2. Invalidate cache — không set giá trị mới
  await this.redis.unlink(`cache:employee:${id}`);

  // 3. Nếu có các cache phụ thuộc, invalidate hết
  await this.redis.unlink(`cache:attendance-summary:${id}:*`); // Lua hoặc pipeline
  // Hoặc dùng tag-based invalidation (nâng cao)

  return employee;
}
```

### 1.3 Cache Invalidation Là Bài Toán Khó

> "There are only two hard things in Computer Science: cache invalidation and naming things." — Phil Karlton

**Vấn đề thực tế**: Employee thay đổi department. Cache của:
- `cache:employee:100` → Phải del
- `cache:department:10:employees` → Phai del
- `cache:attendance-summary:100:*` → Nên del

**Giải pháp pragmatic cho timekeeping**:
1. TTL ngắn + accept eventual consistency — đơn giản nhất
2. Event-driven invalidation: update DB → PUBLISH event → subscriber del các cache liên quan
3. Tag-based invalidation (phức tạp): Mỗi cache key được tag, del theo tag

**Rule production**: Dùng TTL ngắn hơn là thêm logic invalidation phức tạp. Dùng invalidation logic chỉ khi consistency requirement rõ ràng.

---

## 2. TTL Strategy

### 2.1 Bảng TTL Timekeeping System

| Cache Key | TTL | Lý Do | Stale OK? |
|---|---|---|---|
| `cache:employee:{id}` | 600 + jitter(0-60)s | Profile ít thay đổi | Yes, 10 phút |
| `cache:department:{id}` | 1800 + jitter(0-120)s | Department ít thay đổi | Yes, 30 phút |
| `cache:shift:{id}` | 3600 + jitter(0-300)s | Shift rất ít thay đổi | Yes, 1 giờ |
| `cache:attendance-summary:{id}:{date}` | 60 + jitter(0-30)s | Update liên tục | Yes, 1-2 phút |
| `cache:work-schedule:{id}:{week}` | 900 + jitter(0-60)s | Thay đổi ít | Yes, 15 phút |
| `session:{token}` | 28800 | 8 giờ session | No |
| `otp:{phone}` | 180 | 3 phút OTP | No |
| `otp:attempt:{phone}` | 900 | 15 phút rate limit | No |
| `rate:login:{id}` | 900 | 15 phút window | No |
| `rate:checkin:{id}` | 60 | 1 phút window | No |
| `lock:checkin:{id}:{date}` | 10 | Short critical section | No |
| `idempotency:checkin:{req_id}` | 86400 | 24 giờ de-dup | No |
| `idempotency:export:{req_id}` | 3600 | 1 giờ export de-dup | No |

### 2.2 TTL Jitter

```typescript
// Không jitter - nguy hiểm
const TTL = 600;
await redis.set(key, value, 'EX', TTL);
// Nếu 2000 employee cache được set lúc 8:00:00,
// tất cả expire lúc 8:10:00 → thundering herd

// Có jitter - an toàn
const BASE_TTL = 600;
const JITTER = Math.floor(Math.random() * 60); // 0-59 giây
await redis.set(key, value, 'EX', BASE_TTL + JITTER);
// Expire trải ra từ 8:10:00 đến 8:10:59 → giảm tải
```

### 2.3 Stale-While-Revalidate Pattern

Trả về data cũ trong khi refresh background — zero latency cho client.

```typescript
async getWithSWR(key: string, fetcher: () => Promise<any>, ttl: number) {
  const raw = await this.redis.get(key);

  if (raw) {
    const { value, expiresAt } = JSON.parse(raw);
    const remainingTtl = expiresAt - Date.now();

    // Nếu còn > 20% TTL, trả về ngay
    if (remainingTtl > ttl * 1000 * 0.2) {
      return value;
    }

    // Còn < 20% TTL: trả về stale, refresh background
    setImmediate(async () => {
      const fresh = await fetcher();
      const payload = JSON.stringify({
        value: fresh,
        expiresAt: Date.now() + ttl * 1000
      });
      await this.redis.set(key, payload, 'EX', ttl + 60);
    });

    return value; // Trả stale, client không đợi
  }

  // Cache miss: fetch và cache
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

**Mô tả**: Một cache key popular expire → hàng trăm request cùng lúc miss cache → tất cả query DB → DB spike → có thể down.

**Bối cảnh timekeeping**: 8:00 AM, 2000 employee check-in, cache attendance-summary expire, 2000 request cùng hit DB.

**Giải pháp 1: TTL Jitter** (xem 2.2) — Đơn giản nhất.

**Giải pháp 2: Mutex per Key (Cache Lock)**

```typescript
async getWithMutex(key: string, fetcher: () => Promise<any>, ttl: number) {
  // Thử lấy từ cache
  const cached = await this.redis.get(key);
  if (cached) return JSON.parse(cached);

  // Cache miss: Acquire mutex
  const lockKey = `lock:cache-rebuild:${key}`;
  const lockToken = uuidv4();
  const acquired = await this.redis.set(lockKey, lockToken, 'NX', 'EX', 10);

  if (acquired) {
    // Process này thắng lock: fetch và set cache
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
    // Process khác đang rebuild: đợi và retry
    await new Promise(resolve => setTimeout(resolve, 50));
    const retried = await this.redis.get(key);
    return retried ? JSON.parse(retried) : await fetcher();
  }
}
```

**Giải pháp 3: Singleflight Pattern**

```typescript
// Chỉ gọi fetcher 1 lần dù có 100 concurrent request
import { SingleFlight } from 'async-singleflight';

private readonly singleFlight = new SingleFlight();

async get(key: string, fetcher: () => Promise<any>, ttl: number) {
  const cached = await this.redis.get(key);
  if (cached) return JSON.parse(cached);

  // Tất cả concurrent request cho cùng key chỉ gọi fetcher 1 lần
  return this.singleFlight.do(key, async () => {
    const data = await fetcher();
    await this.redis.set(key, JSON.stringify(data), 'EX', ttl);
    return data;
  });
}
```

---

### 3.2 Cache Penetration

**Mô tả**: Request query data không tồn tại trong DB (employee ID = 999999) → mỗi request đều miss cache → hit DB → DB scan vô ích.

**Bối cảnh**: Attacker gửi hàng nghìn request với ID không tồn tại.

**Giải pháp 1: Null Caching**

```typescript
const employee = await this.employeeRepo.findOne({ where: { id } });

if (employee) {
  await this.redis.set(cacheKey, JSON.stringify(employee), 'EX', 600);
} else {
  // Cache kết quả null với TTL ngắn
  await this.redis.set(cacheKey, '__NULL__', 'EX', 30); // 30 giây
}

// Khi đọc:
const raw = await this.redis.get(cacheKey);
if (raw === '__NULL__') return null; // Biết là null, không query DB
if (raw) return JSON.parse(raw);
```

**Giải pháp 2: Bloom Filter**

```
Khi start: Load tất cả valid employee IDs vào Bloom Filter
Request employee 999999 → Bloom Filter trả về: "Chắc chắn không tồn tại"
→ Trả về 404 ngay, không query cache, không query DB
```

Redis có RedisBloom module hỗ trợ native Bloom Filter. Tuy nhiên, có thể dùng `redis-bloom` hoặc implement ở application layer với `bloom-filters` npm package.

**Giải pháp 3: Input Validation** — Đơn giản nhất cho timekeeping:
- Employee ID phải là số nguyên, trong range hợp lệ
- Validate trước khi hit cache/DB

---

### 3.3 Cache Avalanche

**Mô tả**: Nhiều cache key expire cùng lúc → tất cả request hit DB cùng lúc → DB quá tải.

**Khác stampede**: Stampede = 1 key popular. Avalanche = nhiều key cùng expire.

**Bối cảnh**: Morning cache warming set tất cả 2000 employee cache với cùng TTL = 600s → lúc 8:10 tất cả expire cùng lúc.

**Giải pháp**:

```typescript
// 1. TTL Jitter khi set cache
const ttl = baseTtl + Math.floor(Math.random() * maxJitter);

// 2. Staggered cache warming
async warmCache(employees: Employee[]): Promise<void> {
  const pipeline = this.redis.pipeline();
  employees.forEach((emp, index) => {
    // Stagger TTL: mỗi employee thêm 1 giây
    const ttl = 600 + (index % 300); // Spread qua 5 phút
    pipeline.set(`cache:employee:${emp.id}`, JSON.stringify(emp), 'EX', ttl);
  });
  await pipeline.exec();
}

// 3. Cache warming trước khi expire
// Monitor TTL, warm lại khi còn 20%
```

---

### 3.4 Hot Key

**Mô tả**: Một key bị request quá nhiều (vd: "employee:1" của CEO, "config:global"). Redis single-threaded → hot key trở thành bottleneck.

**Bối cảnh**: 2000 employee xem lịch làm việc chung → tất cả đọc cùng 1 `cache:work-schedule:global`.

**Giải pháp 1: Local In-Memory Cache (L1 Cache)**

```typescript
import NodeCache from 'node-cache';

private localCache = new NodeCache({ stdTTL: 10 }); // 10 giây

async getHotKey(key: string): Promise<any> {
  // L1: In-memory cache (process local)
  const local = this.localCache.get(key);
  if (local !== undefined) return local;

  // L2: Redis
  const redis = await this.redis.get(key);
  if (redis) {
    const value = JSON.parse(redis);
    this.localCache.set(key, value); // Cache ở local
    return value;
  }

  // L3: DB
  const data = await this.fetchFromDB(key);
  await this.redis.set(key, JSON.stringify(data), 'EX', 600);
  this.localCache.set(key, data);
  return data;
}
```

**Giải pháp 2: Key Sharding**

```typescript
// Thay vì 1 hot key, dùng N keys (shard theo request)
const SHARD_COUNT = 10;
const shardId = Math.floor(Math.random() * SHARD_COUNT);
const key = `cache:work-schedule:global:shard:${shardId}`;

// Mỗi request đọc từ 1 shard ngẫu nhiên
// Load trải đều trên 10 key thay vì 1
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

  // SET NX PX = atomic "set if not exists" với TTL milliseconds
  const result = await this.redis.set(lockKey, token, 'NX', 'PX', ttlMs);

  return result === 'OK' ? token : null;
}
```

### 4.2 Release Lock (Phải Dùng Lua)

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

**Tại sao phải Lua?** DEL không kiểm tra value → có thể xóa lock của process khác:

```
T1: Process A acquire lock, token = "abc", TTL = 10s
T2: Process A bị timeout (GC pause, slow query)
T3: Lock expire (hết 10s)
T4: Process B acquire lock, token = "xyz"
T5: Process A resume, gọi DEL lock:checkin:100:2026-07-08
    → Xóa lock của Process B!
T6: Process C cũng acquire lock → 2 process cùng run critical section
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

  // Watchdog: extend TTL trong khi job chạy
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
  10000 // 10 giây lock TTL
);
```

### 4.4 Rules Distributed Lock

| Rule | Lý Do |
|---|---|
| Luôn có TTL (PX/EX) | Tránh lock mãi mãi nếu process crash |
| Luôn có unique token | Tránh xóa lock của process khác |
| Release bằng Lua | Atomic check-and-delete |
| TTL ngắn (5-30s) | Tránh lock bị giữ quá lâu |
| Watchdog nếu job dài hơn TTL | Tránh lock expire giữa chừng |
| DB constraint là chốt cuối | Lock Redis có thể fail → DB UNIQUE constraint bắt lỗi |

```sql
-- PostgreSQL: DB là chốt cuối cùng
CREATE TABLE attendances (
  employee_id BIGINT NOT NULL,
  work_date DATE NOT NULL,
  check_in_time TIMESTAMP,
  CONSTRAINT uq_attendance_emp_date UNIQUE (employee_id, work_date)
);
```

**Redis lock tránh duplicate call. DB UNIQUE constraint tránh duplicate data.** Cả hai cần có mặt.

---

## 5. Idempotency Key

### 5.1 Vấn Đề Idempotency

```
Client gửi POST /checkin
Server xử lý: 200ms
Network timeout ở client (300ms) → Client không nhận response
Client retry: gửi lại POST /checkin
Server xử lý lần 2: → Duplicate check-in!
```

**Giải pháp**: Idempotency key — mỗi request có unique request_id, server de-dup.

### 5.2 Idempotency Flow

```
Client gửi:
  POST /checkin
  Headers: X-Idempotency-Key: req-550e8400-e29b-41d4-a716-446655440000
  Body: { employee_id: 100, timestamp: "2026-07-08T08:00:00" }

Server:
  1. key = idempotency:checkin:req-550e8400-...
  2. SET key "processing" NX EX 86400
     - "OK" → Request mới, xử lý tiếp
     - nil → Key tồn tại, là duplicate

  3. Nếu là duplicate:
     - GET key
     - Nếu "processing" → Trả về 202 Accepted (đang xử lý)
     - Nếu JSON result → Trả về kết quả cũ (idempotent response)

  4. Xử lý: insert PostgreSQL
  5. SET key JSON.stringify(result) EX 86400  (lưu kết quả)

Client nhận kết quả, dù retry nhiều lần, response giống nhau.
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
      return next.handle(); // Không có key → pass through
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
      // Trả về kết quả cũ
      return of(JSON.parse(stored));
    }

    // Xử lý request mới
    return next.handle().pipe(
      tap(async (response) => {
        // Lưu kết quả để trả cho retry sau
        await this.redis.set(redisKey, JSON.stringify(response), 'EX', 86400);
      }),
      catchError(async (error) => {
        // Xóa key nếu lỗi để client có thể retry
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

  // Lua script atomic: INCR + set TTL nếu là request đầu tiên
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

**Điểm yếu Fixed Window**: Boundary issue.

```
Window: 0-60 giây. Limit: 10 req/phút
T=59.0s: 10 requests → đã đầy window
T=60.0s: Window reset
T=60.5s: 10 requests mới → OK theo window

Thực tế: 20 requests trong 1.5 giây (59.0 đến 60.5)
```

### 6.2 Sliding Window (Chính Xác Hơn)

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

    -- Xóa request cũ ngoài window
    redis.call('ZREMRANGEBYSCORE', key, '-inf', window_start)

    -- Đếm request trong window
    local count = redis.call('ZCARD', key)

    if count < limit then
      -- Cho phép: thêm request mới
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

### 6.3 Rate Limit cho Các Endpoint Timekeeping

```typescript
// Cấu hình rate limit
const RATE_LIMITS = {
  'checkin': { limit: 3, windowSeconds: 60 },     // 3 check-in/phút
  'login': { limit: 5, windowSeconds: 900 },       // 5 lần login fail/15 phút
  'otp-send': { limit: 3, windowSeconds: 300 },    // 3 OTP/5 phút
  'export-report': { limit: 2, windowSeconds: 60 }, // 2 export/phút
  'api-global': { limit: 100, windowSeconds: 60 }, // 100 req/phút mỗi employee
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
// Trả về headers cho client biết trạng thái
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
    'MAXLEN', '~', '50000',  // Giữ tối đa ~50k messages
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

### 7.3 Consumer với Retry Logic

```typescript
@Injectable()
export class AttendanceStreamConsumer implements OnModuleInit {
  private readonly STREAM_KEY = 'stream:attendance-events';
  private readonly GROUP_NAME = 'notification-workers';
  private readonly CONSUMER_NAME = `worker-${process.pid}`;
  private readonly MAX_RETRIES = 3;
  private readonly PENDING_TIMEOUT_MS = 30000; // 30 giây

  async onModuleInit() {
    // Tạo consumer group nếu chưa có
    try {
      await this.redis.xgroup(
        'CREATE', this.STREAM_KEY, this.GROUP_NAME, '$', 'MKSTREAM'
      );
    } catch (e) {
      if (!e.message.includes('BUSYGROUP')) throw e;
    }

    // Start consuming
    this.consume();
    this.processPending(); // Xử lý message pending (bị drop)
  }

  private async consume() {
    while (true) {
      try {
        const messages = await this.redis.xreadgroup(
          'GROUP', this.GROUP_NAME, this.CONSUMER_NAME,
          'COUNT', '10',
          'BLOCK', '5000', // Wait 5 giây
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
        await new Promise(r => setTimeout(r, 1000)); // Tránh tight loop
      }
    }
  }

  private async handleMessage(id: string, data: Record<string, string>) {
    try {
      await this.sendNotification(data);

      // Xử lý thành công → ACK
      await this.redis.xack(this.STREAM_KEY, this.GROUP_NAME, id);
      this.logger.log(`ACK message ${id}`);

    } catch (error) {
      this.logger.warn(`Failed to process ${id}`, error);
      // Không ACK → message ở trong pending list
      // Sau PENDING_TIMEOUT_MS, processPending() sẽ claim và retry
    }
  }

  // Xử lý các message pending (chưa được ACK sau timeout)
  private async processPending() {
    while (true) {
      await new Promise(r => setTimeout(r, 10000)); // Chạy mỗi 10 giây

      try {
        // Lấy message đã pending > 30s
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
            // Vượt quá số lần retry → chuyển sang DLQ
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
    // Tìm message cụ thể trong pending
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
// Monitor DLQ định kỳ
@Cron('*/5 * * * *') // Mỗi 5 phút
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
    // Reprocess manually hoặc push lại vào main stream
    await this.redis.xadd('stream:attendance-events', 'MAXLEN', '~', '50000', '*', ...Object.entries(data).flat());
    await this.redis.xdel('stream:attendance-events:dlq', id);
  }
}
```

---

## 8. Full Check-in Flow — Bài Tập Lớn

### 8.1 Architecture tổng thể

```
POST /checkin
  Body: { employee_id: 100, timestamp: "2026-07-08T08:00:00" }
  Headers: X-Idempotency-Key: req-uuid-abc

         ↓
  [1] Rate Limit Guard
      rate:checkin:100 → INCR → limit 3/phút
      Nếu vượt → 429 Too Many Requests

         ↓
  [2] Idempotency Check
      idempotency:checkin:req-uuid-abc
      SET NX → "processing"
      Nếu tồn tại → trả kết quả cũ hoặc 202

         ↓
  [3] Acquire Distributed Lock
      lock:checkin:100:2026-07-08 → SET NX PX 10000 token123
      Nếu fail → 409 Conflict (đang xử lý)

         ↓
  [4] Write PostgreSQL
      INSERT INTO attendances (employee_id, work_date, check_in_time)
      ON CONFLICT (employee_id, work_date) DO NOTHING
      → DB UNIQUE constraint là chốt cuối

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

  [9] Worker nhận event từ stream
      → Send notification to employee
      → Update realtime dashboard
      → XACK

  [10] Nếu worker fail → pending list
       → Sau 30s → XCLAIM → retry
       → Sau 3 lần fail → DLQ
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
      // Xóa idem key để client có thể retry
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

### Kiến thức

- [ ] Giải thích Cache-Aside pattern: read flow, write/invalidation flow
- [ ] Giải thích tại sao DEL cache sau update, không SET giá trị mới
- [ ] Nêu 3 cache failure modes: stampede, penetration, avalanche
- [ ] Giải thích Singleflight và Mutex-per-key giải quyết stampede
- [ ] Giải thích Null Caching giải quyết penetration
- [ ] Giải thích Distributed Lock: acquire, hold, release đúng cách
- [ ] Giải thích tại sao DB UNIQUE constraint vẫn cần bên cạnh Redis lock
- [ ] Giải thích Idempotency Key flow cho retry-safe API
- [ ] So sánh Fixed Window vs Sliding Window rate limit
- [ ] Mô tả Queue/Retry/DLQ flow với Streams

### Thực hành

- [ ] Implement Cache-Aside với TTL jitter và null caching
- [ ] Implement distributed lock acquire/release (với Lua)
- [ ] Implement rate limiter (fixed window, Lua atomic)
- [ ] Implement idempotency key interceptor
- [ ] Implement stream producer (XADD)
- [ ] Implement stream consumer với retry (XREADGROUP + XACK)
- [ ] Implement pending processor (XAUTOCLAIM)
- [ ] Implement DLQ move logic

---

## 10. Production Pitfalls Day 3

### P1: Invalidation Race Condition

```
T1: Request A: GET employee:100 → cache miss → query DB
T2: Request B: UPDATE employee:100 + DEL cache:employee:100
T3: Request A: SET cache:employee:100 = OLD DATA (data trước update!)
```

**Hậu quả**: Cache bị poison với data cũ.

**Giải pháp**:
- Chấp nhận eventual consistency với TTL ngắn (most cases)
- Dùng versioning: cache key bao gồm version number → update data → increment version → cache cũ tự bỏ qua

---

### P2: Lock TTL Quá Ngắn

```
Job cần 15 giây nhưng lock TTL = 10 giây
T=10s: Lock expire
T=11s: Process B acquire lock cùng resource
T=12s: Process A và B cùng run critical section
```

**Giải pháp**: Watchdog / lock renewal. Hoặc tính TTL dựa trên expected job duration x 2.

---

### P3: Idempotency Key Xung Đột

```
Client A: X-Idempotency-Key: req-123
Client B: X-Idempotency-Key: req-123  (trùng lặp — bug phía client)
```

**Giải pháp**: Idempotency key phải unique per user. Key = `idempotency:{user_id}:{request_uuid}` thay vì chỉ `{request_uuid}`.

---

### P4: Rate Limit Bypass với Distributed System

```
Load balancer → Server 1: rate:login:100 = 5 (đã đầy)
             → Server 2: rate:login:100 = 0 (server khác, cache khác?)
```

**Giải pháp**: Rate limit phải ở **centralized Redis**, không phải local cache. Tất cả server phải dùng cùng Redis instance/cluster.

---

### P5: Stream Consumer Không XACK → Pending Phình To

```
Worker xử lý message → App crash trước khi XACK
→ Message trong pending list
→ Không có pending processor → Message stuck mãi mãi
→ Sau nhiều tháng: 100k pending entries → memory issue
```

**Giải pháp**: Bắt buộc phải có pending processor với XAUTOCLAIM. Monitor XPENDING count. Alert nếu > ngưỡng.

---

### P6: Cache Warming Cold Start

```
Redis restart (mới, rỗng)
→ 2000 employee check-in lúc 8h
→ Tất cả 2000 request cache miss
→ 2000 DB query cùng lúc
→ DB quá tải
```

**Giải pháp**: Cache warming sau restart:
```typescript
@OnEvent('app.started')
async warmCacheOnStart() {
  const employees = await this.employeeRepo.find({ where: { status: 'active' } });
  await this.cacheService.warmCache(employees); // Staggered pipeline set
}
```

---

## 11. Senior Review Questions Day 3

**Q1**: Tại sao DEL cache sau DB update, không SET cache với giá trị mới?

> **Answer**: Với concurrent writes, nếu 2 process cùng update DB sau đó SET cache, process nào SET sau sẽ override. Nếu update của process A kết thúc trước B, nhưng SET cache của A lấy sau B (do race condition), cache sẽ lưu giá trị cũ của A. DEL an toàn hơn vì nó buộc request tiếp theo query fresh data từ DB. Trade-off: DEL → 1 cache miss sau update. SET → risk poison cache.

**Q2**: Mô tả full distributed lock lifecycle trong check-in API.

> **Answer**: (1) SET NX PX 10000 với unique UUID token. (2) Nếu nil → duplicate request đang chạy → reject. (3) Chạy critical section (INSERT PostgreSQL). (4) Watchdog renew TTL nếu job > 10s. (5) Release: Lua check token == current value → DEL. (6) Nếu process crash → TTL tự expire, process khác có thể acquire sau 10s. (7) DB UNIQUE constraint là safety net cuối cùng.

**Q3**: Cache penetration có thể gây DOS không? Giải quyết thế nào?

> **Answer**: Có. Attacker gửi 10k request với employee IDs không tồn tại → 10k DB queries. Giải pháp: (1) Input validation — ID phải hợp lệ trước khi hit cache/DB. (2) Null caching: cache kết quả null với TTL 30s → request kế tiếp hit cache. (3) Bloom Filter — trước khi query cache, kiểm tra ID có khả năng tồn tại không (false negative = 0, false positive nhỏ).

**Q4**: Sliding window rate limit chính xác hơn fixed window thế nào? Trade-off là gì?

> **Answer**: Fixed window có boundary issue: 3 req cuối phút 1 + 3 req đầu phút 2 = 6 req trong 2 giây. Sliding window dùng Sorted Set với score = timestamp, xóa request ngoài window → luôn chính xác. Trade-off: Sliding window tốn O(1) space per request (Sorted Set member), fixed window chỉ O(1) total per key. Với limit 10 req/phút, sliding window giữ tối đa 10 members per key — chấp nhận được.

**Q5**: Nếu Stream consumer bị crash giữa lúc xử lý 100 message, điều gì xảy ra?

> **Answer**: 100 message ở trong pending list, chưa được XACK. Consumer khác (hoặc consumer restart) phải dùng XAUTOCLAIM để claim lại các message đã pending > timeout (vd: 30s). Delivery count tăng. Nếu delivery count > MAX_RETRIES → move sang DLQ. Do vậy: (1) Bắt buộc có pending processor chạy định kỳ. (2) Monitor XPENDING count. (3) Monitor DLQ. (4) Business logic trong consumer phải idempotent (xử lý 2 lần cũng OK).

**Q6**: Tại sao nói "Redis lock là tranh, DB constraint là chốt"?

> **Answer**: Redis lock giảm tải và tránh race condition trong normal case. Nhưng Redis có thể fail: network partition, Redis restart, lock TTL expire sớm, bug trong release code. Nếu chỉ dựa vào Redis lock, 2 process có thể ghi duplicate khi Redis gặp vấn đề. DB UNIQUE constraint là atomic, durable, không có race condition → là lớp bảo vệ cuối cùng. Pattern chuẩn: Redis lock + DB constraint + idempotent operation.

---

## Appendix: Decision Tree

### Nên Dùng Pattern Nào?

```
Cần giảm DB load cho read-heavy?
  → Cache-Aside

Data hay thay đổi và cần consistency cao?
  → TTL ngắn (30-60s) + accept eventual consistency
  → hoặc event-driven invalidation

Nhiều concurrent request cho cùng 1 cache key?
  → Mutex per key hoặc Singleflight

Query data không tồn tại?
  → Null caching + Input validation

Nhiều cache keys expire cùng lúc?
  → TTL jitter

Chống duplicate write?
  → Distributed lock (Redis) + DB UNIQUE constraint

API có thể bị retry?
  → Idempotency key

Cần giới hạn số request?
  → Fixed window (đơn giản)
  → Sliding window (chính xác)

Cần xử lý event background?
  → Redis Streams với consumer group

Event quan trọng, không được mất?
  → Streams với XACK + retry + DLQ
  
Event nhẹ, mất được?
  → Pub/Sub (đơn giản hơn)
```

---

*Day 3 hoàn thành. Next: Day 4 — Redis Persistence, Replication & Cluster.*
