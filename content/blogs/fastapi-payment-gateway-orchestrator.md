---
title: "Building an Idempotent Payment Orchestrator with FastAPI: Surviving Double-Charges and Dropped Webhooks"
date: 2026-07-10T10:00:00+05:45
slug: fastapi-payment-gateway-orchestrator
categories:
  - Backend
  - FinTech
tags:
  - FastAPI
  - Python
  - SQLAlchemy
  - Alembic
  - Payment Gateway
  - Distributed Systems
  - FinTech
summary: "A battle-tested guide to architecting a resilient payment orchestrator in FastAPI, handling double-spend race conditions, idempotency keys, and timing-safe webhook verification."
description: "Learn how to build a production payment orchestrator in FastAPI with async SQLAlchemy, idempotency keys, state machines, and timing-safe HMAC-SHA256 verification."
author: "Rishav Dahal"
keywords: ["FastAPI Payment Gateway", "Idempotency Keys Python", "eSewa Khalti Orchestrator", "Async SQLAlchemy", "FinTech Architecture"]
cover:
  image: "/images/fastapi-payment-orchestrator.jpg"
  alt: "High-tech 3D isometric architecture of a digital payment gateway orchestrator with FastAPI"
  caption: "Production payment orchestration pipeline with cryptographic validation and idempotency locks"
  relative: false
showtoc: true
draft: false
---

> 📦 **Open Source Repository**: The complete codebase, architecture schemas, and implementation files for this system are available on GitHub at [**rishav-dahal/Payment**](https://github.com/rishav-dahal/Payment).

Anyone who has built checkout systems in production knows the sinking feeling in your stomach when a user messages support: *"Your app charged my wallet twice, but my order says failed!"*

When dealing with payment gateways (like eSewa, Khalti, Stripe, or local bank switch APIs), the network is your worst enemy:
- A user on a spotty mobile connection taps "Pay Now", sees a loading spinner for 6 seconds, gets impatient, and taps it two more times.
- The payment gateway successfully debits the user's balance, but their cellular connection drops right before redirecting back to your success callback.
- A webhook arrives out of order, or the gateway retries the webhook five times in three seconds due to high latency.

If your backend treats payment requests naively, you will double-bill customers, create orphaned orders, or suffer catastrophic reconciliation nightmares at the end of the month.

Here is the exact architectural pattern I use to build a bulletproof payment gateway orchestrator in **FastAPI** with **async SQLAlchemy**, **Alembic**, and **Redis-backed idempotency**.

---

## The Core Problem: The Unreliable Network Triangle

```
[Mobile App / Client]
       │
   1. Initiates Payment ($50)
       ▼
[FastAPI Backend] ──── 2. Calls Gateway API ───► [Payment Gateway (eSewa/Khalti)]
       │                                                      │
(Connection Drops!)                                   3. Debits $50 from User
       │                                                      │
   4. User clicks "Pay" again!                                │
       ▼                                                      │
[FastAPI Backend] ──── 5. Duplicate Transaction? ◄────────────┘
```

If you don't enforce **idempotency** at the API gateway layer, Step 5 generates a completely new transaction ID, sends the user to another payment session, and drains their wallet a second time.

---

## 1. The Idempotency Layer: Guaranteeing Exactly-Once Execution

An **Idempotency Key** is a unique client-generated UUID attached to the `Idempotency-Key` HTTP header. 

If the client retries the request with the same key, your server must return the exact same response as the first successful attempt without re-executing payment side effects.

Here is the custom FastAPI dependency we use:

```python
# core/idempotency.py
import json
from fastapi import Request, HTTPException, status, Depends
from redis.asyncio import Redis

async def get_redis():
    client = Redis.from_url("redis://localhost:6379/0", decode_responses=True)
    try:
        yield client
    finally:
        await client.aclose()

async def enforce_idempotency(request: Request, redis: Redis = Depends(get_redis)):
    key = request.headers.get("Idempotency-Key")
    if not key:
        # Require idempotency key on all mutating financial endpoints
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="Idempotency-Key header is strictly required for payment operations."
        )

    cache_key = f"idempotency:{key}"
    
    # Try to set lock with 120s TTL
    # NX=True ensures only the first request acquires the lock
    acquired = await redis.set(cache_key, "IN_PROGRESS", nx=True, ex=120)
    
    if not acquired:
        cached_value = await redis.get(cache_key)
        if cached_value == "IN_PROGRESS":
            raise HTTPException(
                status_code=status.HTTP_409_CONFLICT,
                detail="A payment request with this idempotency key is currently being processed."
            )
        # Return previously cached response payload
        return json.loads(cached_value)

    # Return key for post-processing cache storage
    return {"key": key, "is_new": True}
```

---

## 2. Relational Schema & State Machine

Payments must never jump arbitrarily between states. We enforce a strict state machine:

$$\text{INITIATED} \longrightarrow \text{PENDING} \longrightarrow \begin{cases} \text{SETTLED} \\ \text{FAILED} \\ \text{EXPIRED} \end{cases}$$

Using SQLAlchemy 2.0 async syntax:

```python
# models/payment.py
import enum
import uuid
from decimal import Decimal
from datetime import datetime
from sqlalchemy import String, Numeric, DateTime, Enum, ForeignKey
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship

class Base(DeclarativeBase):
    pass

class PaymentStatus(str, enum.Enum):
    INITIATED = "INITIATED"
    PENDING = "PENDING"
    SETTLED = "SETTLED"
    FAILED = "FAILED"
    EXPIRED = "EXPIRED"

class PaymentTransaction(Base):
    __tablename__ = "payment_transactions"

    id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
    idempotency_key: Mapped[str] = mapped_column(String(64), unique=True, index=True)
    user_id: Mapped[str] = mapped_column(String(64), index=True)
    gateway_provider: Mapped[str] = mapped_column(String(32)) # 'esewa', 'khalti', 'stripe'
    
    # Always store monetary amounts with strict numeric precision
    amount: Mapped[Decimal] = mapped_column(Numeric(12, 2))
    currency: Mapped[str] = mapped_column(String(3), default="NPR")
    
    status: Mapped[PaymentStatus] = mapped_column(
        Enum(PaymentStatus), default=PaymentStatus.INITIATED, index=True
    )
    gateway_reference_id: Mapped[str | None] = mapped_column(String(128), index=True)
    
    created_at: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow)
    updated_at: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
```

---

## 3. Atomic Database Row Locking (`SELECT ... FOR UPDATE`)

When a webhook arrives at the exact same millisecond that a user manually refreshes their confirmation page, two parallel threads attempt to update the same row.

Without row-level locks, both threads read `PENDING`, both mark it `SETTLED`, and both fire your "fulfill order" background task (e.g., granting digital credits or issuing tickets twice).

In async SQLAlchemy, we lock the row during status updates:

```python
# services/payment.py
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession
from models.payment import PaymentTransaction, PaymentStatus

async def settle_transaction(
    session: AsyncSession, 
    transaction_id: str, 
    gateway_ref: str
) -> PaymentTransaction:
    async with session.begin():
        # SELECT ... FOR UPDATE locks this exact row until commit/rollback
        stmt = (
            select(PaymentTransaction)
            .where(PaymentTransaction.id == transaction_id)
            .with_for_update()
        )
        result = await session.execute(stmt)
        tx = result.scalar_one_or_none()

        if not tx:
            raise ValueError("Transaction not found")

        # Idempotent state transition guard
        if tx.status == PaymentStatus.SETTLED:
            # Already processed! Return safely without re-triggering fulfillment
            return tx

        if tx.status != PaymentStatus.PENDING and tx.status != PaymentStatus.INITIATED:
            raise ValueError(f"Cannot settle transaction in {tx.status} state")

        tx.status = PaymentStatus.SETTLED
        tx.gateway_reference_id = gateway_ref
        
        # Trigger internal fulfillment logic here
        await trigger_order_fulfillment(tx)

    return tx
```

---

## 4. The Dangerous Flaw: Webhook Timing Attacks

When the payment gateway sends a webhook notification, it includes a signature header (e.g., HMAC-SHA256).

Here is the mistake 90% of developers make:
```python
# DANGEROUS: Susceptible to timing attacks!
if incoming_signature == computed_signature:
    process_payment()
```

The standard `==` operator in Python compares strings character by character and exits on the first mismatch. An attacker measuring microscopic differences in server response latency (tens of microseconds) can iteratively deduce the valid cryptographic signature!

**The Production Fix**: Always use constant-time comparisons:
```python
import hmac

def verify_webhook_signature(payload: bytes, secret: str, received_signature: str) -> bool:
    expected_signature = hmac.new(
        secret.encode("utf-8"),
        payload,
        hashlib.sha256
    ).hexdigest()

    # Compares in constant time regardless of where mismatches occur
    return hmac.compare_digest(expected_signature, received_signature)
```

---

## 5. The FastAPI Payment Endpoint

Putting it all together into a clean, modern async route:

```python
# api/payments.py
from fastapi import APIRouter, Depends, Header, HTTPException, status
from sqlalchemy.ext.asyncio import AsyncSession
from pydantic import BaseModel, Field
from decimal import Decimal
import uuid

router = APIRouter(prefix="/payments", tags=["payments"])

class CreatePaymentRequest(BaseModel):
    amount: Decimal = Field(gt=0, decimal_places=2)
    gateway: str = Field(regex="^(esewa|khalti|stripe)$")

@router.post("/initiate")
async def initiate_payment(
    body: CreatePaymentRequest,
    idempotency_data: dict = Depends(enforce_idempotency),
    session: AsyncSession = Depends(get_db)
):
    # If this request was already processed, return cached payload immediately
    if not idempotency_data.get("is_new"):
        return idempotency_data

    idem_key = idempotency_data["key"]

    tx = PaymentTransaction(
        id=uuid.uuid4(),
        idempotency_key=idem_key,
        user_id="usr_98124",
        gateway_provider=body.gateway,
        amount=body.amount,
        status=PaymentStatus.INITIATED
    )
    session.add(tx)
    await session.commit()

    # Call gateway adapter (e.g. eSewa v2 or Khalti intent API)
    gateway_payload = await gateway_client.create_charge_intent(tx)

    response_data = {
        "transaction_id": str(tx.id),
        "status": tx.status.value,
        "payment_url": gateway_payload["redirect_url"]
    }

    # Store in Redis so immediate retries get the identical response
    await redis.set(f"idempotency:{idem_key}", json.dumps(response_data), ex=86400)

    return response_data
```

---

## Hard-Earned Lessons

1. **Never trust redirect URLs for settlement**: When a customer finishes paying on eSewa or Khalti and gets redirected back to your `success_url`, **never** mark the order as paid based solely on that browser redirect. The user could tamper with query parameters. Only settle when your backend receives the direct server-to-server webhook or you poll the gateway's status verification API.
2. **Handle gateway downtime with Dead Letter Queues (DLQ)**: If your database is under maintenance when a webhook hits, your server returns a 500 error. Most gateways retry for a few hours and then stop forever. Push raw webhook payloads to RabbitMQ or an SQS Dead Letter Queue before processing them so you can replay missed events.
3. **Log raw payloads**: Store the verbatim string of every incoming webhook in an `audit_logs` table before parsing it. When a reconciliation dispute arises, having the unparsed payload with the exact gateway timestamp is your best defense.


---

## 🛠️ GitHub Repository & Next Steps

The complete open-source source code and architecture discussed in this guide are publicly available:

- **Project Repository**: [Payment Microservice & Gateway Orchestrator on GitHub](https://github.com/rishav-dahal/Payment)
- **Developer Profile**: [@rishav-dahal](https://github.com/rishav-dahal)

If you're building a similar system or encounter edge cases in your deployment, feel free to star the repo, file an issue, or submit an optimization pull request!
