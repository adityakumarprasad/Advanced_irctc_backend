# 🚪 API Gateway — Complete Deep Dive

> **Gateway pura project ka darwaza hai. Koi bhi request bina Gateway ke andar nahi aa sakti.**

---

## Table of Contents

1. [API Gateway Kya Hai?](#api-gateway-kya-hai)
2. [package.json](#packagejson)
3. [.env.example](#envexample)
4. [config/index.js — Configuration](#configindexjs--configuration)
5. [config/logger.js — Winston Logger](#configloggerjs--winston-logger)
6. [config/redis.js — Redis Singleton](#configredisjs--redis-singleton)
7. [utils/error.js — Custom Error Classes](#utilserrorjs--custom-error-classes)
8. [middlewares/cors.middleware.js — CORS](#middlewarescorsmiddlewarejs--cors)
9. [middlewares/req.middleware.js — Request Logger](#middlewaresreqmiddlewarejs--request-logger)
10. [middlewares/auth.middleware.js — JWT Auth](#middlewaresauthmiddlewarejs--jwt-auth)
11. [middlewares/rateLimiting.middleware.js — Rate Limiter](#middlewaresratelimitingmiddlewarejs--rate-limiter)
12. [middlewares/error.middleware.js — Error Handler](#middlewareserrormiddlewarejs--error-handler)
13. [middlewares/notFound.middleware.js — 404 Handler](#middlewaresnotfoundmiddlewarejs--404-handler)
14. [services/proxy.js — Circuit Breaker + Proxy](#servicesproxy--circuit-breaker--proxy)
15. [routes/index.js — Route Definitions](#routesindexjs--route-definitions)
16. [index.js — Entry Point](#indexjs--entry-point)
17. [Execution Flow](#execution-flow)

---

## API Gateway Kya Hai?

### Simple Definition
API Gateway ek **reverse proxy** hai jo client aur backend services ke beech me baithta hai. Think of it as a **security guard + receptionist**:
- Security guard: "Tumhara ID card (JWT) dikhao"
- Receptionist: "Tumhara kaam User Service me hoga, waha bhejta hoon"

### Why API Gateway?

```mermaid
graph TB
    subgraph "Without Gateway ❌"
        C1["Client"] --> US1["User Service:4001"]
        C1 --> BS1["Booking Service:4005"]
        C1 --> PS1["Payment Service:4006"]
        Note1["Client ko har service ka\naddress yaad rakhna padega.\nAuth logic har service me duplicate."]
    end

    subgraph "With Gateway ✅"
        C2["Client"] --> GW["API Gateway:4000"]
        GW --> US2["User Service:4001"]
        GW --> BS2["Booking Service:4005"]
        GW --> PS2["Payment Service:4006"]
        Note2["Client sirf ek address jaanta hai.\nAuth centralized.\nRate limiting centralized."]
    end
```

### Responsibilities

| Responsibility | Kya karta hai? | Kaise? |
|---|---|---|
| **Authentication** | JWT verify karta hai | `auth.middleware.js` |
| **Rate Limiting** | Zyada requests block karta hai | Redis sorted sets |
| **Circuit Breaking** | Down service pe requests rok deta hai | `CircuitBreaker` class |
| **Proxying** | Request correct service ko forward karta hai | Axios-based proxy |
| **CORS** | Browser security headers manage karta hai | `cors` package |
| **Request Logging** | Har request ka log rakhta hai | Winston logger |
| **Error Handling** | Sab errors ko consistent format me return karta hai | Error middleware |

---

## package.json

```json
{
  "name": "api-gateway",
  "version": "1.0.0",
  "main": "src/index.js",
  "scripts": {
    "start": "node src/index.js",
    "dev": "nodemon src/index.js"
  },
  "type": "commonjs",
  "dependencies": {
    "axios": "^1.13.4",
    "cookie-parser": "^1.4.7",
    "cors": "^2.8.6",
    "crypto": "^1.0.1",
    "dotenv": "^17.2.4",
    "express": "^5.2.1",
    "helmet": "^8.1.0",
    "ioredis": "^5.9.2",
    "jsonwebtoken": "^9.0.3",
    "morgan": "^1.10.1",
    "winston": "^3.19.0"
  },
  "devDependencies": {
    "nodemon": "^3.1.11"
  }
}
```

### Dependency Explanation

| Package | Kya hai? | Kyu lagaya? |
|---|---|---|
| `express` | Web framework | HTTP server + routing |
| `axios` | HTTP client | Downstream services ko requests forward karne ke liye |
| `cors` | CORS middleware | Browser cross-origin requests allow karne ke liye |
| `helmet` | Security headers | XSS, clickjacking, etc. se protection |
| `ioredis` | Redis client | Rate limiting ke liye Redis se baat karna |
| `jsonwebtoken` | JWT library | Access tokens verify karna |
| `cookie-parser` | Cookie parser | HTTP cookies parse karna (JWT cookies ke liye) |
| `dotenv` | Env loader | `.env` file se environment variables load karna |
| `winston` | Logger | Structured logging |
| `morgan` | HTTP logger | Development me request logs dikhana |
| `crypto` | Cryptography | Node.js built-in (listed for explicit dependency tracking) |
| `nodemon` | Dev tool | File change pe auto-restart |

### `"type": "commonjs"` Kya Hai?

Node.js me do module systems hain:
1. **CommonJS**: `require()` / `module.exports` — Default, synchronous
2. **ES Modules**: `import` / `export` — Modern, async, tree-shakeable

Yeh project CommonJS use karta hai. `"type": "commonjs"` explicitly batata hai.

---

## .env.example

```env
PORT=4000
NODE_ENV=development

ALLOWED_ORIGINS=http://localhost:3000
REDIS_URL=redis://:your-password@localhost:6379

JWT_ACCESS_SECRET=your_access_secret
JWT_REFRESH_SECRET=your_refresh_secret
ACCESS_TOKEN_EXP=15m
REFRESH_TOKEN_EXP=7d
ACCESS_TOKEN_EXP_SEC=900
REFRESH_TOKEN_EXP_SEC=604800

RATE_LIMIT_WINDOW_MS=900000
RATE_LIMIT_MAX_REQUESTS=100

USER_SERVICE_URL=http://localhost:4001
SEARCH_SERVICE_URL=http://localhost:4002
BOOKING_SERVICE_URL=http://localhost:4003
...

SERVICE_TIMEOUT_MS=60000
CIRCUIT_BREAKER_THRESHOLD=5
CIRCUIT_BREAKER_TIMEOUT=60000
```

### Every Variable Explained

| Variable | Value | Explanation |
|---|---|---|
| `PORT` | 4000 | Gateway kis port pe chale |
| `NODE_ENV` | development | Production me "production" hoga — error details hide hongi |
| `ALLOWED_ORIGINS` | http://localhost:3000 | CORS — kaun se domains se requests allow hain |
| `REDIS_URL` | redis://:pass@host:port | Redis connection string — rate limiting ke liye |
| `JWT_ACCESS_SECRET` | random string | Access token sign/verify karne ka secret |
| `JWT_REFRESH_SECRET` | different random string | Refresh token sign/verify karne ka alag secret |
| `ACCESS_TOKEN_EXP` | 15m | Access token 15 minutes me expire hoga |
| `REFRESH_TOKEN_EXP` | 7d | Refresh token 7 days me expire hoga |
| `ACCESS_TOKEN_EXP_SEC` | 900 | 15 minutes in seconds (cookie maxAge ke liye) |
| `REFRESH_TOKEN_EXP_SEC` | 604800 | 7 days in seconds |
| `RATE_LIMIT_WINDOW_MS` | 900000 | 15 minute window for rate limiting |
| `RATE_LIMIT_MAX_REQUESTS` | 100 | Max 100 requests per window |
| `SERVICE_TIMEOUT_MS` | 60000 | 60 seconds timeout for downstream service calls |
| `CIRCUIT_BREAKER_THRESHOLD` | 5 | 5 failures → circuit opens |
| `CIRCUIT_BREAKER_TIMEOUT` | 60000 | 60 seconds wait before retrying (HALF_OPEN) |

### Why Two JWT Secrets?

**Security best practice**: Access aur Refresh tokens ke secrets alag hone chahiye. Agar ek leak ho jaye, dusra safe rahe.

### Why Seconds AND String Duration?

`ACCESS_TOKEN_EXP = "15m"` — JWT library ke `expiresIn` option ke liye
`ACCESS_TOKEN_EXP_SEC = 900` — Cookie `maxAge` ke liye (milliseconds chahiye, seconds × 1000)

---

## config/index.js — Configuration

### File Purpose
Saare environment variables ko ek structured `config` object me organize karta hai. Application me `process.env` directly access karne ki jagah `config.PORT` use karte hain.

### Code — Line by Line

```javascript
const config = {
     PORT: process.env.PORT || 4000,
```
- `process.env.PORT` — Environment variable read karo
- `|| 4000` — Agar set nahi hai toh default 4000 use karo
- **Production reason**: Docker/Kubernetes me port environment variable se milta hai

```javascript
     SERVICE_NAME: require('../../package.json').name,
```
- `package.json` se service name padho — "api-gateway"
- **Why?**: Logger me service name automatically aata hai, hardcode nahi karna padta
- `require()` JSON files ko automatically parse kar deta hai

```javascript
     NODE_ENV: process.env.NODE_ENV || 'development',
```
- Application mode: development, staging, production
- **Effect**: Development me detailed error logs, production me minimal

```javascript
     REDIS_URL: process.env.REDIS_URL,
```
- Yaha default value NAHI di — **required variable** hai
- Lekin validation neeche nahi hai iske liye (bug? ya optional?)

```javascript
     JWT_ACCESS_SECRET: process.env.JWT_ACCESS_SECRET,
     JWT_REFRESH_SECRET: process.env.JWT_REFRESH_SECRET,
```
- JWT secrets — **critically important**
- Inke bina tokens verify nahi ho sakte

```javascript
     SERVICES: {
          USER_SERVICE_URL: process.env.USER_SERVICE_URL || 'http://localhost:4001',
          SEARCH_SERVICE_URL: process.env.SEARCH_SERVICE_URL || 'http://localhost:4002',
          ADMIN_SERVICE_URL: process.env.ADMIN_SERVICE_URL || 'http://localhost:4003',
          NOTIFICATION_SERVICE_URL: process.env.NOTIFICATION_SERVICE_URL || 'http://localhost:4004',
          BOOKING_SERVICE_URL: process.env.BOOKING_SERVICE_URL || 'http://localhost:4005',
          PAYMENT_SERVICE_URL: process.env.PAYMENT_SERVICE_URL || 'http://localhost:4006',
          INVENTORY_SERVICE_URL: process.env.INVENTORY_SERVICE_URL || 'http://localhost:4007'
     },
```
- **Service Registry** — Har downstream service ka address
- Default values localhost hain development ke liye
- Production/Docker me actual URLs set honge

```javascript
     RATE_LIMIT_WINDOW_MS: parseInt(process.env.RATE_LIMIT_WINDOW_MS || '900000', 10),
     RATE_LIMIT_MAX_REQUESTS: parseInt(process.env.RATE_LIMIT_MAX_REQUESTS || '100', 10),
```
- `parseInt(..., 10)` — String ko decimal number me convert karo
- Radix `10` explicitly dena best practice hai (octal confusion avoid)
- Environment variables hamesha STRING hote hain, isliye parse karna padta hai

```javascript
     CIRCUIT_BREAKER_THRESHOLD: parseInt(process.env.CIRCUIT_BREAKER_THRESHOLD || '5', 10),
     CIRCUIT_BREAKER_TIMEOUT: parseInt(process.env.CIRCUIT_BREAKER_TIMEOUT || '60000', 10),
};
```
- Circuit breaker config — kitni failures pe circuit open ho, kitna wait kare

```javascript
const requiredConfig = ['JWT_ACCESS_SECRET', 'JWT_REFRESH_SECRET'];

requiredConfig.forEach((key) => {
     if (!config[key]) {
          throw new Error(`Missing required environment variable: ${key}`);
     }
});
```
- **Fail-Fast Pattern** — Agar critical environment variables missing hain toh application start hi nahi hoga
- Better than silently failing during a request
- **Interview Answer**: "Fail-fast is better than fail-silently. Application ko startup pe crash hona chahiye agar configuration galat hai, rather than at 3 AM when a customer tries to log in."

### Interview Questions

**Q: Why not read process.env directly everywhere?**
A: "Centralized config object provides: (1) Default values in one place (2) Validation at startup (3) Type conversion (string → number) (4) IDE autocomplete (5) Easy mocking in tests."

**Q: Why `parseInt(x, 10)` and not just `Number(x)`?**
A: "`Number('')` returns `0`, `parseInt('', 10)` returns `NaN`. `Number('10px')` returns `NaN`, `parseInt('10px', 10)` returns `10`. Depends on what behavior you want."

---

## config/logger.js — Winston Logger

```javascript
const winston = require('winston');
const {config} = require('.');

const logger = winston.createLogger({
     level: config.LOG_LEVEL,
     defaultMeta: {service: config.SERVICE_NAME},
     format: winston.format.combine(
          winston.format.timestamp(),
          winston.format.printf(({level, message, timestamp, service}) => {
               return `[${timestamp}] [${level}] [${service}]: ${message}`
          })
     ),
     transports: [new winston.transports.Console()]
})

module.exports = logger;
```

### 📚 Winston Complete Teaching

**Kya hai Winston?**
Winston Node.js ka most popular logging library hai. `console.log()` ki jagah use karte hain because:

1. **Log Levels**: `error` > `warn` > `info` > `debug` — Production me sirf `error` aur `warn` dikhao
2. **Structured Output**: Timestamp, service name, level — sab automatically
3. **Multiple Transports**: Console, File, HTTP, CloudWatch — ek jagah configure karo
4. **Metadata**: Extra data attach karo (userId, requestId, etc.)

### Line by Line

```javascript
const winston = require('winston');
```
- Winston library import

```javascript
const {config} = require('.');
```
- Same folder ka `index.js` import (config object)
- `.` means current directory — Node.js automatically `index.js` pick karta hai

```javascript
const logger = winston.createLogger({
```
- Logger instance create karo

```javascript
     level: config.LOG_LEVEL,
```
- Minimum log level set karo
- `LOG_LEVEL = 'info'` → `debug` messages hide honge
- Production me `'warn'` set karo → sirf warnings aur errors dikhenge

**Log Level Hierarchy:**
```
error: 0   ← Most important
warn:  1
info:  2
http:  3
verbose: 4
debug: 5   ← Most detailed
silly: 6
```

```javascript
     defaultMeta: {service: config.SERVICE_NAME},
```
- Har log message me automatically `service: 'api-gateway'` add hoga
- **Why?**: Centralized logging me (CloudWatch, DataDog) pata chalega kaun si service ka log hai

```javascript
     format: winston.format.combine(
          winston.format.timestamp(),
```
- Multiple formatters combine karo
- `timestamp()` — ISO timestamp add karo (e.g., `2026-06-26T10:30:45.123Z`)

```javascript
          winston.format.printf(({level, message, timestamp, service}) => {
               return `[${timestamp}] [${level}] [${service}]: ${message}`
          })
```
- Custom format define karo
- Output: `[2026-06-26T10:30:45.123Z] [info] [api-gateway]: Server started`
- **Destructuring**: `{level, message, timestamp, service}` — log info object se fields nikalo

```javascript
     transports: [new winston.transports.Console()]
```
- Log kaha bhejne hain — console pe
- Production me add karo: `new winston.transports.File({ filename: 'error.log', level: 'error' })`

### Production Improvements

```javascript
// Production-grade logger example:
transports: [
     new winston.transports.Console(),
     new winston.transports.File({ filename: 'logs/error.log', level: 'error' }),
     new winston.transports.File({ filename: 'logs/combined.log' }),
     // CloudWatch, DataDog, etc.
]
```

---

## config/redis.js — Redis Singleton

```javascript
const Redis = require('ioredis');
const { config } = require('.');
const logger = require('./logger');

class RedisClient {
     static instance;
     static isConnected = false;

     constructor() {
          // prevent direct instantiation
     }

     static getInstance() {
          if (!RedisClient.instance) {
               RedisClient.instance = new Redis(config.REDIS_URL, {
                    retryStrategy: (times) => {
                         const delay = Math.min(times * 50, 2000);
                         return delay;
                    },
                    maxRetriesPerRequest: 3
               })
               RedisClient.setupEventListeners();
          }
          return RedisClient.instance;
     }

     static setupEventListeners() { ... }
     static async closeConnection() { ... }
     static isReady() { return RedisClient.isConnected; }
     static async testConnection() { ... }
}

module.exports = {
     redis: RedisClient.getInstance(),
     RedisClient
}
```

### 📚 Singleton Pattern Complete Teaching

**Kya hai Singleton?**
Ek design pattern jisme class ka **sirf ek instance** banta hai poore application me.

**Why Singleton for Redis?**
- Redis connection expensive hai — TCP connection establish karna, authentication, etc.
- Multiple connections waste resources
- Ek hi connection sab jagah share karo

**How Singleton Works:**

```mermaid
graph TD
    A["File A: require('./redis')"] -->|"Calls getInstance()"| S["RedisClient"]
    B["File B: require('./redis')"] -->|"Calls getInstance()"| S
    C["File C: require('./redis')"] -->|"Calls getInstance()"| S
    S -->|"Returns SAME instance"| R["Single Redis Connection"]
```

### Line by Line

```javascript
static instance;
static isConnected = false;
```
- **Static fields** — Class pe hain, instance pe nahi
- `instance` — Singleton connection store karta hai (initially `undefined`)
- `isConnected` — Connection status track karta hai

```javascript
constructor() {
     // prevent direct instantiation
}
```
- Empty constructor — log ko `new RedisClient()` se instance banana discourage karte hain
- **Note**: JavaScript me truly private constructor nahi ban sakta (unlike Java/C#)

```javascript
static getInstance() {
     if (!RedisClient.instance) {
```
- **Lazy initialization** — Pehli baar call hone pe hi connection create hoga
- Subsequent calls pe same instance return hoga

```javascript
          RedisClient.instance = new Redis(config.REDIS_URL, {
               retryStrategy: (times) => {
                    const delay = Math.min(times * 50, 2000);
                    return delay;
               },
               maxRetriesPerRequest: 3
          })
```
- `new Redis(url, options)` — ioredis connection create karo
- **retryStrategy**: Connection fail hone pe retry logic
  - `times` = number of retries so far
  - `times * 50` = exponential backoff (50ms, 100ms, 150ms...)
  - `Math.min(..., 2000)` = max 2 seconds wait
  - `return delay` — ioredis ko batao kitna wait karke retry karna hai
  - `return null` → retrying band karo
- `maxRetriesPerRequest: 3` — Ek Redis command max 3 baar retry hoga

### Event Listeners

```javascript
static setupEventListeners() {
     RedisClient.instance.on('connect', () => {
          RedisClient.isConnected = true;
          logger.info("Connected to Redis");
     })

     RedisClient.instance.on('error', (error) => {
          RedisClient.isConnected = false;
          logger.error("Redis connection error", error);
     })
     // ... close, reconnecting, ready, end
}
```

**Redis Connection Lifecycle:**
```mermaid
stateDiagram-v2
    [*] --> connecting: new Redis()
    connecting --> connect: TCP connected
    connect --> ready: AUTH + SELECT done
    ready --> close: Connection lost
    close --> reconnecting: retryStrategy
    reconnecting --> connect: Reconnected
    close --> end: Manual quit / max retries
    ready --> end: client.quit()
```

### Export Strategy

```javascript
module.exports = {
     redis: RedisClient.getInstance(),
     RedisClient
}
```
- `redis` — Pre-created instance, ready to use: `redis.get('key')`
- `RedisClient` — Class export for advanced usage (e.g., `RedisClient.closeConnection()`)

**Module Cache**: Node.js `require()` result cache karta hai. Toh `require('./redis')` kitni baar bhi call karo, `RedisClient.getInstance()` sirf ek baar run hoga.

### Interview Questions

**Q: Singleton vs Dependency Injection — kaun better hai?**
A: "Singleton simple hai par testing me problem hai (hard to mock). DI testable hai par boilerplate zyada. Small projects me Singleton chalega, large projects me DI framework (like InversifyJS) better."

**Q: Redis connection pool kyu nahi use kiya?**
A: "ioredis internally multiplexing karta hai — ek TCP connection pe multiple commands pipeline karta hai. Pool ki zarurat nahi padti unless bahut high throughput hai."

---

## utils/error.js — Custom Error Classes

```javascript
class AppError extends Error {
     constructor(message, statusCode, code) {
          super(message);
          this.statusCode = statusCode;
          this.code = code;
          Error.captureStackTrace(this, this.constructor);
     }
}

class BadRequestError extends AppError {
     constructor(message, code = 'BAD_REQUEST') {
          super(message, 400, code);
     }
}

class UnauthorizedError extends AppError { ... } // 401
class ForbiddenError extends AppError { ... }    // 403
class NotFoundError extends AppError { ... }     // 404
class ConflictError extends AppError { ... }     // 409
class TooManyRequestsError extends AppError { ... } // 429
class InternalServerError extends AppError { ... }  // 500
class ServiceUnavailableError extends AppError { ... } // 503
class GatewayTimeoutError extends AppError { ... }     // 504
```

### 📚 Error Handling Pattern — Complete Teaching

**Problem bina custom errors ke:**
```javascript
// Bad approach
throw new Error("User not found");
// Error middleware ko kaise pata chalega ki 404 bhejni hai ya 500?
```

**Solution — Custom Error Hierarchy:**
```mermaid
classDiagram
    Error <|-- AppError
    AppError <|-- BadRequestError
    AppError <|-- UnauthorizedError
    AppError <|-- ForbiddenError
    AppError <|-- NotFoundError
    AppError <|-- ConflictError
    AppError <|-- TooManyRequestsError
    AppError <|-- InternalServerError
    AppError <|-- ServiceUnavailableError
    AppError <|-- GatewayTimeoutError

    class AppError {
        +statusCode: number
        +code: string
        +message: string
    }
```

### Line by Line

```javascript
class AppError extends Error {
```
- JavaScript native `Error` class extend kar rahe hain
- `instanceof Error` → `true` (stack trace, message automatically milega)

```javascript
     constructor(message, statusCode, code) {
          super(message);
```
- `super(message)` — Parent `Error` class ka constructor call karo
- `message` property set ho jayegi

```javascript
          this.statusCode = statusCode;
          this.code = code;
```
- Custom properties add karo
- `statusCode` — HTTP status code (400, 401, 404, etc.)
- `code` — Application-specific error code (e.g., 'TOKEN_EXPIRED', 'SEATS_UNAVAILABLE')

```javascript
          Error.captureStackTrace(this, this.constructor);
```
- **V8-specific method** (Node.js = V8 engine)
- Stack trace me AppError ka constructor nahi dikhega, sirf actual error location dikhega
- **Without**: Stack trace me `AppError → BadRequestError → actualCode` dikhta
- **With**: Stack trace me directly `actualCode` dikhta — cleaner debugging

### HTTP Status Codes Cheat Sheet

| Code | Class | Meaning | When Use? |
|---|---|---|---|
| 400 | BadRequestError | Invalid request data | Validation fail |
| 401 | UnauthorizedError | Not authenticated | Missing/invalid JWT |
| 403 | ForbiddenError | Not authorized | Wrong role/permissions |
| 404 | NotFoundError | Resource not found | User/booking doesn't exist |
| 409 | ConflictError | State conflict | Duplicate booking, seat already taken |
| 429 | TooManyRequestsError | Rate limit exceeded | Too many API calls |
| 500 | InternalServerError | Server bug | Unhandled errors |
| 503 | ServiceUnavailableError | Service down | Circuit breaker open |
| 504 | GatewayTimeoutError | Upstream timeout | Service response too slow |

### TooManyRequestsError — Extra Property

```javascript
class TooManyRequestsError extends AppError {
     constructor(message, retryAfter = 60, code = 'TOO_MANY_REQUESTS') {
          super(message, 429, code);
          this.retryAfter = retryAfter;
     }
}
```
- `retryAfter` — Client ko batao kitne seconds baad retry kare
- Response header `Retry-After` me set hota hai
- **Rate limiting standard** (RFC 6585)

---

## middlewares/auth.middleware.js — JWT Authentication

```javascript
const jwt = require('jsonwebtoken');
const { config } = require('../config');
const { UnauthorizedError } = require('../utils/error');
const logger = require('../config/logger');

function requireAuth(req, res, next) {
     try {
          let accessToken;

          // 1. Try Authorization header (service-to-service / mobile clients)
          const authHeader = req.headers.authorization;
          if (authHeader && authHeader.startsWith('Bearer ')) {
               accessToken = authHeader.split(' ')[1];
          }

          // 2. Fall back to httpOnly cookie (browser clients)
          if (!accessToken && req.cookies) {
               accessToken = req.cookies.accessToken;
          }

          if (!accessToken) {
               throw new UnauthorizedError('Authorization token missing');
          }

          // Verify access token
          const payload = jwt.verify(accessToken, config.JWT_ACCESS_SECRET);

          if (!payload.id) {
               throw new UnauthorizedError('Invalid token payload');
          }

          // Attach user context to request for downstream services
          req.user = { id: payload.id };

          // Add user ID to headers for proxied requests
          req.headers['x-user-id'] = payload.id.toString();

          logger.debug(`User ${payload.id} authenticated successfully`);
          next();
     } catch (err) {
          if (err.name === 'TokenExpiredError') {
               return next(new UnauthorizedError('Access token expired', 'TOKEN_EXPIRED'));
          }
          if (err.name === 'JsonWebTokenError') {
               return next(new UnauthorizedError('Invalid access token', 'TOKEN_INVALID'));
          }
          return next(err);
     }
}

module.exports = { requireAuth };
```

### 📚 JWT Complete Teaching

**Kya hai JWT (JSON Web Token)?**

JWT ek **stateless authentication token** hai. Stateless matlab server ko session store nahi karna padta — token me hi saari information hoti hai.

**JWT Structure:**
```
eyJhbGciOiJIUzI1NiJ9.eyJpZCI6IjEyMzQifQ.signature
|_____ Header _____| |____ Payload ____| |_ Signature _|
```

- **Header**: `{"alg": "HS256"}` — Algorithm used
- **Payload**: `{"id": "1234", "iat": 1234, "exp": 1234}` — Data
- **Signature**: `HMAC-SHA256(header + "." + payload, secret)` — Tamper-proof

**JWT Analogy**: 
> Movie ticket ki tarah — ticket pe likhaa hai kaun si movie, kaun si seat, kab ki hai. Theater wala sirf ticket check karta hai, database me nahi dekhta. Agar ticket modify kare toh watermark invalid ho jayega (signature mismatch).

### Line by Line

```javascript
function requireAuth(req, res, next) {
```
- Express middleware — 3 parameters (req, res, next)
- `requireAuth` — naam se clear hai ki authentication required hai

```javascript
     let accessToken;
     const authHeader = req.headers.authorization;
     if (authHeader && authHeader.startsWith('Bearer ')) {
          accessToken = authHeader.split(' ')[1];
     }
```
- **Authorization Header** se token extract karo
- Format: `Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...`
- `split(' ')[1]` — "Bearer" ke baad ka part lo

**Why "Bearer" prefix?**
RFC 6750 standard. Different auth schemes exist: Basic, Bearer, Digest, etc.

```javascript
     if (!accessToken && req.cookies) {
          accessToken = req.cookies.accessToken;
     }
```
- **Fallback to Cookie** — Browser clients httpOnly cookie bhejte hain
- Mobile apps Authorization header bhejte hain
- **Dual support** — dono types of clients support karo

**Why httpOnly cookie > localStorage?**
- `httpOnly` cookie me JavaScript access NAHI kar sakta → **XSS attack proof**
- localStorage me token store karne pe XSS se churi ho sakta hai

```javascript
     const payload = jwt.verify(accessToken, config.JWT_ACCESS_SECRET);
```
- `jwt.verify()` — Token verify karo:
  1. Signature check — Token tampered toh nahi?
  2. Expiry check — Token expired toh nahi?
  3. Decode — Payload extract karo

- **Throws** `TokenExpiredError` ya `JsonWebTokenError` on failure

```javascript
     req.user = { id: payload.id };
```
- Decoded user info ko `req` object pe attach karo
- Downstream code me `req.user.id` se access kar sakte hain

```javascript
     req.headers['x-user-id'] = payload.id.toString();
```
- **Critical for microservice communication**
- Gateway downstream service ko request proxy karta hai
- Downstream service ko user ID chahiye — header me bhej do
- `.toString()` — Header values must be strings

```javascript
     } catch (err) {
          if (err.name === 'TokenExpiredError') {
               return next(new UnauthorizedError('Access token expired', 'TOKEN_EXPIRED'));
          }
```
- **Specific error mapping** — Different JWT errors → Different response codes
- `TOKEN_EXPIRED` → Frontend ko pata chalega ki refresh token use karna hai
- `TOKEN_INVALID` → Frontend ko pata chalega ki re-login karna hai

### Authentication Flow

```mermaid
sequenceDiagram
    participant C as Client
    participant GW as API Gateway
    participant US as User Service

    Note over C,GW: 1. Login Flow
    C->>GW: POST /api/users/auth/login
    GW->>US: Forward (no auth needed)
    US-->>GW: {accessToken, refreshToken}
    GW-->>C: Set-Cookie: accessToken, refreshToken

    Note over C,GW: 2. Authenticated Request
    C->>GW: GET /api/users/user/profile Cookie: accessToken=eyJ...
    GW->>GW: requireAuth → jwt.verify()
    GW->>GW: Set header x-user-id=123
    GW->>US: GET /user/profile x-user-id: 123
    US-->>GW: User profile data
    GW-->>C: 200 OK

    Note over C,GW: 3. Token Expired
    C->>GW: GET /api/users/user/profile
    GW->>GW: jwt.verify() → TokenExpiredError
    GW-->>C: 401 {code: "TOKEN_EXPIRED"}
    C->>GW: POST /api/users/auth/refresh Cookie: refreshToken=eyJ...
    GW->>US: Forward
    US-->>GW: New accessToken + refreshToken
    GW-->>C: Set-Cookie (new tokens)
```

---

## middlewares/rateLimiting.middleware.js — Rate Limiter

### 📚 Rate Limiting Complete Teaching

**Kya hai Rate Limiting?**
Ek mechanism jo ek user/IP ki requests ko ek time window me limit karta hai.

**Why?**
1. **DDoS protection** — Millions of requests se server overload na ho
2. **Brute force prevention** — OTP guessing, password cracking
3. **Fair usage** — Ek user sab bandwidth na khaye
4. **Cost control** — Downstream API calls (SendGrid, Razorpay) ka cost control

### Sliding Window Algorithm

```mermaid
graph LR
    subgraph "Sliding Window (15 min)"
        T1["Request at T+0"] 
        T2["Request at T+2min"]
        T3["Request at T+5min"]
        T4["Request at T+10min"]
        T5["Request at T+14min"]
    end

    subgraph "Window slides forward"
        direction TB
        W1["At T+16min: T+0 request drops out"]
        W2["New request can come in"]
    end
```

**Redis Sorted Set (ZSET)** use karta hai:
- Score = timestamp
- Member = unique request ID
- `ZREMRANGEBYSCORE` — purane entries remove karo
- `ZADD` — naya entry add karo
- `ZCARD` — count karo
- `EXPIRE` — key ka TTL set karo

### Code — Key Function

```javascript
async function rateLimiter(key, maxRequests, windowMs) {
     const now = Date.now();
     const windowStart = now - windowMs;

     try {
          const pipeline = redis.pipeline();
          pipeline.zremrangebyscore(key, 0, windowStart);    // Remove old entries
          pipeline.zadd(key, now, `${now}-${Math.random()}`); // Add current request
          pipeline.zcard(key);                                 // Count total in window
          pipeline.expire(key, Math.ceil(windowMs / 1000));   // Set key TTL

          const results = await pipeline.exec();
          const requestCount = results[2][1]; // Third command's result

          if (requestCount > maxRequests) {
               // ... return rate limit exceeded
          }

          return { allowed: true, remaining: maxRequests - requestCount };
     } catch (err) {
          // Fail open - allow request if Redis is down
          return { allowed: true, remaining: maxRequests };
     }
}
```

### Redis Pipeline Explained

```javascript
const pipeline = redis.pipeline();
```
- **Pipeline** = Multiple Redis commands ek batch me bhejo
- **Without pipeline**: 4 network roundtrips (4 × ~1ms = ~4ms)
- **With pipeline**: 1 network roundtrip (~1ms)

**Why Atomic?**
- Sabhi commands ek saath execute hote hain
- Koi doosra request beech me interfere nahi kar sakta
- **Race condition** prevent hoti hai

### Three Rate Limiting Strategies

```javascript
// 1. IP-based — Unauthenticated users
function ipRateLimit(options = {}) { ... }
// Key: ratelimit:ip:192.168.1.1

// 2. User-based — Authenticated users (more lenient)
function userRateLimit(options = {}) { ... }
// Key: ratelimit:user:user123

// 3. Endpoint-specific — Sensitive endpoints
function endpointRateLimit(maxRequests, windowMs) { ... }
// Key: ratelimit:endpoint:POST:/api/users/auth/send-otp:192.168.1.1

// 4. Combined — Both IP + User limits
function combinedRateLimit(ipOptions = {}, userOptions = {}) { ... }
```

### Fail-Open Design

```javascript
} catch (err) {
     logger.error('Rate limiter error:', err);
     // Fail open - allow request if Redis is down
     return { allowed: true, remaining: maxRequests };
}
```

**Critical Design Decision:**
- Redis down → Rate limiting skip karo, request allow karo
- Alternative: **Fail-closed** — Redis down → Sab requests block karo
- **Why fail-open?**: Availability > Security for rate limiting. User ko 429 dena worse hai than temporary unprotected state.

### Response Headers

```javascript
res.setHeader('X-RateLimit-Limit', maxRequests);
res.setHeader('X-RateLimit-Remaining', result.remaining);
res.setHeader('X-RateLimit-Reset', new Date(result.resetTime).toISOString());
```
- **Standard headers** — Client ko batao limit kya hai aur kitni remaining hain
- Frontend retry logic me use hota hai

---

## services/proxy.js — Circuit Breaker + Proxy

### 📚 Circuit Breaker Pattern Complete Teaching

**Analogy:**
> Ghar me MCB (Miniature Circuit Breaker) lagta hai. Jab excess current aaye toh MCB trip ho jaata hai — saara circuit band ho jaata hai taaki wiring na jale. Manually reset karna padta hai.

**Software me bhi wahi concept:**
- Jab downstream service baar baar fail ho
- Circuit "open" ho jaata hai
- Requests directly reject ho jaati hain (fast fail)
- Service ko recover hone ka time milta hai
- Kuch time baad ek request bhej ke check karo (half-open)
- Agar success → circuit "close" karo (normal operations resume)

### Circuit Breaker State Machine

```mermaid
stateDiagram-v2
    [*] --> CLOSED: Initial State
    CLOSED --> OPEN: failureCount >= threshold
    OPEN --> HALF_OPEN: timeout elapsed
    HALF_OPEN --> CLOSED: Request succeeds
    HALF_OPEN --> OPEN: Request fails
    
    note right of CLOSED: Normal operation All requests pass through
    note right of OPEN: All requests rejected Fast fail (503)
    note right of HALF_OPEN: One test request allowed Decides next state
```

### Code — CircuitBreaker Class

```javascript
class CircuitBreaker {
     constructor(serviceName, threshold = 5, timeout = 60000) {
          this.serviceName = serviceName;
          this.failureCount = 0;
          this.threshold = threshold;
          this.timeout = timeout;
          this.state = 'CLOSED';
          this.nextAttempt = Date.now();
     }

     async execute(request) {
          if (this.state === 'OPEN') {
               if (Date.now() < this.nextAttempt) {
                    throw new ServiceUnavailableError(
                         `Service ${this.serviceName} is temporarily unavailable.`
                    );
               }
               this.state = 'HALF_OPEN';
          }

          try {
               const response = await request();
               this.onSuccess();
               return response;
          } catch (err) {
               this.onFailure();
               throw err;
          }
     }

     onSuccess() {
          this.failureCount = 0;
          if (this.state === 'HALF_OPEN') {
               this.state = 'CLOSED';
          }
     }

     onFailure() {
          this.failureCount++;
          if (this.failureCount >= this.threshold) {
               this.state = 'OPEN';
               this.nextAttempt = Date.now() + this.timeout;
          }
     }
}
```

### Per-Service Circuit Breakers

```javascript
const circuitBreakers = {
     userService: new CircuitBreaker('user-service'),
     searchService: new CircuitBreaker('search-service'),
     adminService: new CircuitBreaker('admin-service'),
     notificationService: new CircuitBreaker('notification-service'),
     bookingService: new CircuitBreaker('booking-service'),
     paymentService: new CircuitBreaker('payment-service'),
     inventoryService: new CircuitBreaker('inventory-service')
};
```

**Why per-service?**
- User Service down hai toh sirf user requests fail hongi
- Booking Service still kaam karega (agar independently accessible ho)
- **Blast radius minimize** karo

### Proxy Function — createProxy()

```javascript
function createProxy(serviceName, serviceUrl) {
     const circuitBreaker = circuitBreakers[serviceName];

     return async (req, res, next) => {
          try {
               // Extract path: /users/auth/login -> /auth/login
               const pathParts = req.path.split('/').filter(Boolean);
               const servicePath = '/' + pathParts.slice(1).join('/');

               const result = await forwardRequest(
                    serviceUrl,
                    servicePath + (req.url.includes('?') ? req.url.substring(req.url.indexOf('?')) : ''),
                    req.method,
                    req.body,
                    req.headers,
                    circuitBreaker
               );

               // Forward response headers
               Object.keys(result.headers).forEach((key) => {
                    if (!excludeHeaders.includes(key.toLowerCase())) {
                         res.setHeader(key, result.headers[key]);
                    }
               });

               res.status(result.status).json(result.data);
          } catch (err) {
               next(err);
          }
     };
}
```

### Path Transformation

```
Gateway URL:  /api/users/auth/login
                    ↓
After /api strip (route level): /users/auth/login
                    ↓
Path parts:   ['users', 'auth', 'login']
                    ↓
Slice(1):     ['auth', 'login']
                    ↓
Service URL:  http://localhost:4001/auth/login
```

### forwardRequest — The Actual Proxy

```javascript
async function forwardRequest(serviceUrl, path, method, data, headers, circuitBreaker) {
     const requestConfig = {
          method,
          url: `${serviceUrl}${path}`,
          timeout: config.SERVICE_TIMEOUT_MS,
          headers: {
               ...headers,
               host: undefined,           // Remove host header
               'content-length': undefined, // Let axios recalculate
          },
          validateStatus: () => true,  // Don't throw on non-2xx
     };
```

**`validateStatus: () => true`**: 
- Normally Axios throws error on 4xx/5xx responses
- Here we WANT those responses to pass through to the client
- Gateway shouldn't interpret downstream errors — just forward them

**Headers cleanup:**
- `host: undefined` — Remove because downstream service ka host alag hai
- `content-length: undefined` — Body modify ho sakta hai, Axios recalculate karega

### Error Handling

```javascript
     } catch (err) {
          if (err.code === 'ECONNABORTED' || err.code === 'ETIMEDOUT') {
               throw new GatewayTimeoutError(`Request timed out...`);
          }
          if (err.code === 'ECONNREFUSED') {
               throw new ServiceUnavailableError(`Cannot connect to service...`);
          }
          if (err.response) {
               return { status: err.response.status, data: err.response.data, ... };
          }
          throw new ServiceUnavailableError(`Service temporarily unavailable`);
     }
```

**Error Categories:**
- `ECONNABORTED`/`ETIMEDOUT` → 504 Gateway Timeout
- `ECONNREFUSED` → 503 Service Unavailable (service not running)
- `err.response` exists → Service responded with error, forward it
- Other errors → 503 generic

### Where Companies Use Circuit Breaker

| Company | Implementation |
|---|---|
| **Netflix** | Hystrix (Java) — inspired this entire pattern |
| **Amazon** | Circuit breakers between all microservices |
| **Uber** | Custom implementation in Go |
| **This project** | Custom in-memory JavaScript implementation |

**Production improvement**: Use **Redis-backed state** instead of in-memory, so circuit breaker state persists across instances.

---

## routes/index.js — Route Definitions

### Public vs Private Routes

```javascript
// PUBLIC routes — no auth needed
router.post('/users/auth/send-otp', endpointRateLimit(5, 3600000), userServiceProxy);
router.post('/users/auth/login', endpointRateLimit(100, 900000), userServiceProxy);

// PRIVATE routes — auth required
router.get('/users/user/profile', requireAuth, combinedRateLimit(), userServiceProxy);
router.post('/bookings/bookings', requireAuth, endpointRateLimit(5, 60000), bookingServiceProxy);
```

### Rate Limit Strategy Per Endpoint

| Endpoint | Rate Limit | Reasoning |
|---|---|---|
| `POST /send-otp` | 5/hour | OTP = email cost + abuse prevention |
| `POST /verify-otp` | 10/hour | Brute force prevention |
| `POST /login` | 100/15min | Normal login traffic |
| `GET /search/trains` | 60/minute | Frequent search queries |
| `GET /search/autocomplete` | 120/minute | Typed characters trigger this |
| `POST /bookings/bookings` | 5/minute | Prevent booking spam |
| `POST /payments/webhooks/razorpay` | No limit | Razorpay webhook — never block |

---

## index.js — Entry Point

```javascript
require('dotenv').config();
const express = require('express');
const helmet = require('helmet');
const cookieParser = require('cookie-parser');
const morgan = require('morgan');
// ...

const app = express();

app.use(corsMiddleware);
app.use(helmet({ crossOriginOpenerPolicy: false, crossOriginEmbedderPolicy: false }));
app.use(reqLogger);

// Skip JSON parsing for Razorpay webhook — raw bytes needed
app.use((req, res, next) => {
     if (req.path === '/api/payments/webhooks/razorpay') {
          return express.raw({ type: 'application/json', limit: '10mb' })(req, res, next);
     }
     express.json({ limit: '10mb' })(req, res, next);
});
app.use(express.urlencoded({ extended: true, limit: '10mb' }));
app.use(cookieParser());

if (config.NODE_ENV === 'development') {
     app.use(morgan('dev'));
}

app.get('/health', (req, res) => { ... });
app.use('/api', routes);
app.use(notFound);
app.use(errorHandler);
```

### Middleware Order — CRITICAL

```mermaid
graph TD
    A["CORS"] --> B["Helmet (Security Headers)"]
    B --> C["Request Logger"]
    C --> D["Body Parser (JSON/Raw)"]
    D --> E["URL Encoded Parser"]
    E --> F["Cookie Parser"]
    F --> G["Morgan (Dev only)"]
    G --> H["Health Check"]
    H --> I["API Routes"]
    I --> J["404 Not Found"]
    J --> K["Error Handler"]
```

**Order matters because:**
1. CORS first — Browser preflight requests immediately handled
2. Helmet early — Security headers set before any processing
3. Body parser before routes — Routes me `req.body` available ho
4. Error handler LAST — Catches all errors from above middlewares

### Razorpay Webhook — Raw Body

```javascript
app.use((req, res, next) => {
     if (req.path === '/api/payments/webhooks/razorpay') {
          return express.raw({ type: 'application/json', limit: '10mb' })(req, res, next);
     }
     express.json({ limit: '10mb' })(req, res, next);
});
```

**Why raw body for webhooks?**
- Razorpay signature verification requires **exact original bytes**
- `express.json()` parse karke object bana deta hai
- `JSON.stringify(parsedObject)` wapas karoge toh original bytes different ho sakte hain (key ordering, whitespace)
- `express.raw()` Buffer me raw bytes rakhta hai

### Graceful Shutdown

```javascript
const gracefulShutdown = () => {
     logger.info('Received shutdown signal, closing server gracefully...');
     server.close(() => {
          logger.info('Server closed');
          process.exit(0);
     });

     setTimeout(() => {
          logger.error('Forced shutdown after timeout');
          process.exit(1);
     }, 30000);
};

process.on('SIGTERM', gracefulShutdown);
process.on('SIGINT', gracefulShutdown);
```

**Kya hai Graceful Shutdown?**

Jab server band karna ho (deployment, restart), toh:
1. Naye requests accept karna band karo
2. Existing requests ko complete hone do
3. Database connections close karo
4. Phir exit karo

**`SIGTERM`**: Kubernetes/Docker bhejta hai jab pod/container stop karna ho
**`SIGINT`**: Ctrl+C press karne pe

**30 second timeout**: Agar 30 seconds me existing requests complete nahi hue toh force exit karo (stuck connections se protect karo)

### Unhandled Rejection Handler

```javascript
process.on('unhandledRejection', (err) => {
     logger.error('Unhandled Rejection:', err);
     server.close(() => process.exit(1));
});
```

**Kya hai Unhandled Rejection?**
Jab koi Promise reject ho aur koi `.catch()` na ho.

```javascript
// This would cause unhandledRejection:
someAsyncFunction(); // No await, no .catch()
```

**Best Practice**: Log the error aur server gracefully shutdown karo. Process ko silently crash hone se better hai.

---

## Summary — API Gateway Hinglish Me

Bhai, API Gateway poore system ka **security guard + receptionist + traffic controller** hai.

1. **Client** browser se request bhejta hai → `http://localhost:4000/api/users/auth/login`

2. **CORS** check karta hai — "Yeh domain allowed hai kya?" (e.g., `http://localhost:3000`)

3. **Helmet** security headers lagata hai — XSS, clickjacking se protection

4. **Request Logger** log karta hai — "[POST] /api/users/auth/login"

5. **Body Parser** request body parse karta hai — JSON → JavaScript object

6. **Router** dekhta hai kaun sa route match hua → `/api/users/auth/login`

7. **Rate Limiter** check karta hai — "Yeh IP se kitni requests aayi hain?" (Redis sorted set se)

8. Agar **auth required** hai toh → JWT token verify karta hai → `req.headers['x-user-id']` set karta hai

9. **Circuit Breaker** check karta hai — "User Service alive hai kya?"

10. **Proxy** request forward karta hai → `http://localhost:4001/auth/login`

11. Response wapas client ko bhejta hai

12. Agar koi error aaye toh **Error Handler** consistent JSON response bhejta hai

**Memory trick**: "**CHRB-RACP-RE**" — CORS, Helmet, ReqLogger, BodyParser, Route, Auth, CircuitBreaker, Proxy, Response, ErrorHandler

---

> **Next Chapter**: [02 — User Service Deep Dive](./02_user_service.md)
