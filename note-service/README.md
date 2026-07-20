# 📖 IRCTC Backend — Complete Engineering Handbook (Hinglish Edition)

> **"Best backend engineering notes ever written."**
> 
> Yeh handbook tumhe is project ko scratch se likhne ki capability degi. Har file, har line, har design decision — sab explain hai.

---

## 📚 Chapters

| # | Chapter | Pages | Key Topics |
|---|---|---|---|
| **00** | [🏗️ Project Overview & Architecture](./handbook/00_project_overview_and_architecture.md) | ~700 lines | Architecture diagrams, Docker Compose, Kafka topics, DLQ handler, Request lifecycle |
| **01** | [🚪 API Gateway Deep Dive](./handbook/01_api_gateway.md) | ~800 lines | JWT auth, Rate limiting (Redis sorted sets), Circuit Breaker, Proxy, CORS, Helmet, Graceful shutdown |
| **02** | [📝 Booking Service Deep Dive](./handbook/02_booking_service.md) | ~700 lines | **Saga Pattern**, Distributed Lock (Lua), CAS/Optimistic Concurrency, Leader Election, Expiry Job |
| **03** | [💳 Payment Service Deep Dive](./handbook/03_payment_service.md) | ~500 lines | **Strategy + Factory** (Gateway), Razorpay, HMAC signature, Webhook raw body, Refund tracking |
| **04** | [📦 Inventory, 🔍 Search, 📧 Notification, 🔧 Admin](./handbook/04_inventory_search_notification_admin.md) | ~750 lines | `FOR UPDATE NOWAIT`, Segment booking, Elasticsearch queries, Fuzzy search, SendGrid, Event enrichment |
| **05** | [👤 User Service Deep Dive](./handbook/05_user_service.md) | ~600 lines | OTP (HMAC-hashed), bcrypt, JWT Rotation + Reuse Detection, Google OAuth, Device fingerprinting |
| **06** | [🎯 Design Patterns & Interview Mastery](./handbook/06_design_patterns_and_interviews.md) | ~600 lines | 15 patterns summary, 26+ interview Q&A, System design template, Security matrix, Cheat sheet |

**Total**: ~4,500+ lines of production-level documentation

---

## 🎯 How to Use This Handbook

### For Interview Prep
1. Start with **Chapter 06** — Design Patterns & Interview Bank
2. Read **Chapter 00** — Architecture overview for system design rounds
3. Deep dive into **Chapter 02** (Booking) and **Chapter 03** (Payment) — most complex patterns

### For Building From Scratch
1. Read **Chapter 00** — Set up Docker infrastructure
2. Read **Chapter 01** — Build API Gateway first
3. Read **Chapter 05** — Build User Service (auth foundation)
4. Read **Chapter 04** — Build Admin + Inventory (data foundation)
5. Read **Chapter 02** — Build Booking Service (core business logic)
6. Read **Chapter 03** — Build Payment Service (Razorpay integration)
7. Read **Chapter 04** — Build Search + Notification (supporting services)

### For Quick Reference
- **Redis key patterns** → Chapter 06 cheat sheet
- **Kafka topic mapping** → Chapter 00 event registry
- **HTTP status codes** → Chapter 01 error classes
- **Security measures** → Chapter 06 security matrix

---

## 🧠 Key Concepts Covered

### Design Patterns (15)
`Saga` · `Distributed Lock` · `CAS/OCC` · `Strategy` · `Factory` · `Circuit Breaker` · `Token Rotation` · `Event-Driven` · `DLQ` · `Cache-Aside` · `Idempotency` · `Leader Election` · `Singleton` · `Higher-Order Function` · `Reverse Proxy`

### Technologies
`Node.js` · `Express.js` · `PostgreSQL` · `Prisma ORM` · `Redis` · `Apache Kafka` · `Elasticsearch` · `Docker` · `Razorpay` · `JWT` · `bcrypt` · `SendGrid` · `Google OAuth` · `Winston` · `Lua Scripts`

### Distributed Systems
`CAP Theorem` · `Eventual Consistency` · `Idempotency` · `Distributed Transactions` · `Event Sourcing Lite` · `Leader Election` · `Deadlock Prevention` · `Counter Drift` · `Graceful Shutdown`

---

*Created with ❤️ for backend engineers who want to truly understand production systems.*
