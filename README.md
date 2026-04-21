# 💳 PayFlow — Backend Payment System

> A production-grade backend service simulating a digital wallet and payment processing engine, built with financial-grade transaction integrity in mind.

---

## 📌 Project Overview

This project is a backend payment system that simulates wallet and transaction processing. It is designed to mirror real-world fintech infrastructure — with a strong emphasis on **atomicity**, **auditability**, and **idempotency** at every layer of the payment lifecycle.

This project was built to explore how real-world payment systems ensure **transactional integrity**, **reliability**, and **traceability** at scale.

---

## ⚙️ Core Features

| Feature | Description |
|---|---|
| **Auth** | Register, login, JWT-based session management |
| **Wallet** | View balance, top up via simulated payment gateway |
| **Transfer** | Peer-to-peer fund transfer between wallets |
| **Payment** | Outbound payment to merchant (QR / VA / Card) |
| **Transaction History** | Paginated, filterable transaction ledger per user |
| **Audit Log** | Immutable record of every transaction state change |
| **Balance History** | Full ledger of wallet balance changes over time |

---

## 🔄 Transaction Flow

This is the backbone of the entire system. Every payment and transfer goes through this pipeline — no exceptions.

```
User initiates request (PAYMENT or TRANSFER)
        │
        ▼
Validate request (auth + input schema)
        │
        ▼
Check wallet balance (sufficient funds?)
        │
   ┌────┴────┐
   │ FAILED  │ ──────────────────────────────────► Transaction status = FAILED
   │ (insuf) │                                     Audit log entry created
   └─────────┘
        │
   PASSED ▼
Write transaction record (status = PENDING)
        │
        ▼
Lock wallet balance (prevent double-spend)
        │
        ▼
        ├─── if PAYMENT ──► Write to payments table
        │                   Call external gateway (simulated)
        │
        └─── if TRANSFER ─► Write to transfers table
                            Debit sender wallet
                            Credit receiver wallet
        │
        ▼
   ┌────┴────┐         ┌──────────┐
   │ SUCCESS │         │  FAILED  │
   └────┬────┘         └────┬─────┘
        │                   │
        ▼                   ▼
Update wallet balance    Rollback balance lock
Record balance_history   Update tx → FAILED
Update tx → SUCCESS      Audit log entry created
Audit log entry created
        │
        └──────────────────┘
                 ▼
        Return response to client
```

**Key design principles:**
- Every transaction is written as `PENDING` first — never skip this step
- `PAYMENTS` and `TRANSFERS` are **detail tables** — `TRANSACTIONS` is always the parent
- Balance locks prevent race conditions in concurrent requests
- All state transitions are recorded in `audit_logs` — non-negotiable
- `balance_history` records every debit/credit with before & after snapshot
- Failed transactions are **never silently dropped** — always logged

---

## 🧠 System Architecture

```
┌─────────────────────────────────────────────────────┐
│                    API Layer                        │
│         REST Controllers (Spring Boot)              │
└──────────────────────┬──────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────┐
│                  Service Layer                      │
│      Business Logic · Transaction Orchestration     │
│      Idempotency Check · Balance Lock               │
└────────────┬─────────────────────────┬──────────────┘
             │                         │
┌────────────▼───────────┐   ┌─────────▼──────────────┐
│   Repository Layer     │   │  External Integration  │
│   JPA / Hibernate      │   │  Payment Gateway Mock  │
│   PostgreSQL           │   │  (REST / Callback)     │
└────────────────────────┘   └────────────────────────┘
```

The system follows a **modular monolith architecture**:
- **Controller layer** handles API requests and input validation
- **Service layer** manages business logic and transaction orchestration
- **Repository layer** interacts with the database via JPA
- **Integration layer** communicates with external payment providers

---

## 🔗 External Integration (Simulated)

The system simulates integration with an external payment gateway.

- Payment requests are sent to a mock external service
- Responses determine final transaction status (`SUCCESS` / `FAILED`)
- External reference IDs are stored in the `payments` table for reconciliation

This reflects real-world payment system behavior where services communicate with third-party providers — a core concern in integration platform services.

---

## ⚡ Concurrency & Consistency

To ensure safe concurrent transactions:

- Wallet balance updates are executed within **database transactions** (ACID)
- **Row-level locking** prevents double-spending when multiple requests hit the same wallet simultaneously
- **Idempotency keys** (`reference_no`) ensure requests are not processed multiple times — even on client retries

---

## 🧪 Failure Handling

The system explicitly handles failure scenarios:

| Scenario | Behavior |
|---|---|
| Insufficient balance | Immediate transaction failure — no lock acquired |
| External payment failure | Transaction rolled back, status set to `FAILED` |
| Duplicate request | Detected via `reference_no` — second request rejected cleanly |
| Unexpected error | Exception caught, audit log written, status finalized |

No transaction is ever silently ignored.

---

## 🗄️ Database Design

> See `docs/erd.png` for the full Entity Relationship Diagram.

### Architecture Logic

```
USERS
  └── WALLETS
        └── BALANCE_HISTORY
  └── TRANSACTIONS  ← parent of all financial events
        ├── PAYMENTS       (detail: merchant payment)
        ├── TRANSFERS      (detail: peer-to-peer)
        └── AUDIT_LOGS
```

`TRANSACTIONS` = parent. `PAYMENTS` / `TRANSFERS` = detail.
This is the standard fintech pattern: query one table for all history, join to detail only when needed.

---

### Tables

#### `users`
| Column | Type | Notes |
|---|---|---|
| `id` | UUID | Primary key |
| `name` | VARCHAR(100) | |
| `email` | VARCHAR(100) | Unique |
| `password_hash` | VARCHAR(255) | bcrypt |
| `created_at` | TIMESTAMP | |
| `updated_at` | TIMESTAMP | |

#### `wallets`
| Column | Type | Notes |
|---|---|---|
| `id` | UUID | Primary key |
| `user_id` | UUID | FK → users.id |
| `balance` | DECIMAL(18,2) | Current spendable balance |
| `currency` | VARCHAR(3) | Default: IDR |
| `created_at` | TIMESTAMP | |
| `updated_at` | TIMESTAMP | |

> 1 user = 1 wallet

#### `transactions`
| Column | Type | Notes |
|---|---|---|
| `id` | UUID | Primary key |
| `reference_no` | VARCHAR(50) | Unique — e.g. `TRX-20260421-0001` |
| `user_id` | UUID | FK → users.id |
| `type` | ENUM | `TRANSFER`, `PAYMENT`, `TOPUP` |
| `amount` | DECIMAL(18,2) | |
| `status` | ENUM | `PENDING`, `SUCCESS`, `FAILED` |
| `created_at` | TIMESTAMP | |
| `updated_at` | TIMESTAMP | |

> `reference_no` is the idempotency key. If a client retries a timed-out request, the system will not double-process it.

#### `payments`
| Column | Type | Notes |
|---|---|---|
| `id` | UUID | Primary key |
| `transaction_id` | UUID | FK → transactions.id |
| `merchant_name` | VARCHAR(100) | |
| `external_reference` | VARCHAR(100) | Gateway reference ID |
| `payment_method` | ENUM | `QR`, `VA`, `CARD` |
| `status` | ENUM | `PENDING`, `SUCCESS`, `FAILED` |
| `created_at` | TIMESTAMP | |

> Detail table for merchant payments only.

#### `transfers`
| Column | Type | Notes |
|---|---|---|
| `id` | UUID | Primary key |
| `transaction_id` | UUID | FK → transactions.id |
| `sender_wallet_id` | UUID | FK → wallets.id |
| `receiver_wallet_id` | UUID | FK → wallets.id |
| `created_at` | TIMESTAMP | |

> Detail table for peer-to-peer wallet transfers only.

#### `audit_logs`
| Column | Type | Notes |
|---|---|---|
| `id` | UUID | Primary key |
| `transaction_id` | UUID | FK → transactions.id |
| `action` | ENUM | `CREATED`, `UPDATED`, `FAILED` |
| `description` | TEXT | Human-readable log message |
| `created_at` | TIMESTAMP | Immutable — no updates allowed |

> Append-only table. Never update, never delete.

#### `balance_history`
| Column | Type | Notes |
|---|---|---|
| `id` | UUID | Primary key |
| `wallet_id` | UUID | FK → wallets.id |
| `amount_before` | DECIMAL(18,2) | Balance before change |
| `amount_after` | DECIMAL(18,2) | Balance after change |
| `change_type` | VARCHAR(50) | e.g. `TOPUP`, `DEBIT`, `CREDIT` |
| `created_at` | TIMESTAMP | |

> Lets you reconstruct a wallet's balance at any point in time without replaying all transactions.

---

### Entity Relationships

```
users        (1) ──── (1)   wallets
users        (1) ──── (N)   transactions
wallets      (1) ──── (N)   transfers        [as sender]
wallets      (1) ──── (N)   transfers        [as receiver]
wallets      (1) ──── (N)   balance_history
transactions (1) ──── (0,1) payments
transactions (1) ──── (0,1) transfers
transactions (1) ──── (N)   audit_logs
```

---

## 🛠️ Tech Stack

| Layer | Technology |
|---|---|
| **Runtime** | Java (Spring Boot) |
| **Database** | PostgreSQL |
| **ORM** | Spring Data JPA / Hibernate |
| **Auth** | JWT + Refresh Token rotation |
| **Cache / Lock** | Redis *(balance lock + idempotency check)* |
| **Queue** | RabbitMQ / Spring AMQP *(async payment processing)* |
| **API Docs** | Swagger / OpenAPI 3.0 |
| **Testing** | JUnit 5 + Mockito |

---

## 🚀 Getting Started

```bash
# Clone the repo
git clone https://github.com/your-username/payflow-backend.git
cd payflow-backend

# Setup environment
cp .env.example .env

# Run database migrations
./mvnw flyway:migrate

# Start development server
./mvnw spring-boot:run
```

---

## 📁 Project Structure

```
src/
└── main/
    └── java/
        └── com/payflow/
            ├── modules/
            │   ├── auth/
            │   ├── wallet/
            │   ├── transaction/
            │   ├── payment/
            │   ├── transfer/
            │   └── audit/
            ├── common/
            │   ├── exception/
            │   ├── middleware/
            │   └── response/
            └── config/
resources/
└── db/
    └── migration/        ← Flyway migration scripts
docs/
└── erd.png
```

---

## 📝 License

MIT — see `LICENSE` for details.

---

> *Built with the mindset: if it's not in the audit log, it didn't happen.*
