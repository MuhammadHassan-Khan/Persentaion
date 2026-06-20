# AegisFlow AI — Backend API (Member 6) Presentation Guide

This is your personal prep document. Read it top to bottom once, then use it as a script while rehearsing out loud. The goal isn't to memorize code — it's to be able to explain **what each piece does and why it exists** in plain language, and to survive follow-up questions.

---

## PART 1 — The 60-Second Elevator Pitch (memorize this)

> "AegisFlow AI is a fraud-detection API that online stores plug into their checkout. When a customer places an order, the store's server sends us the transaction details. We run it through three AI analysis agents in parallel — one checks the transaction itself, one checks how the user behaved on the page, one checks the device and network — combine their scores, run a safety/compliance check, and send back a decision: **approve, block, step-up (ask for OTP), or review** — all within about 200 milliseconds. My part is the backend API layer: the actual doors the outside world knocks on, the security that checks who's knocking, and the system that makes sure no merchant can ever see another merchant's data."

Say this without looking at notes. Everything else in this doc supports this one paragraph.

---

## PART 2 — The Whole System, In Plain English (context you need even though it's not your part)

Think of it in **5 layers**, like a pipeline:

1. **Merchant's checkout page** — has a small JavaScript file (`aegisflow.js`) silently watching how the user types, moves the mouse, etc.
2. **Your layer — the API** — receives the request, checks "is this a real paying customer (merchant) and are they allowed to make this call right now?"
3. **The Orchestrator** — the "brain" that fires off 3 specialist AI agents at the same time:
   - **Transaction Agent** — checks card velocity (same card used too many times too fast?), known fraud patterns, runs a machine learning model.
   - **Behavioral Agent** — did the user paste the card number instead of typing it? Did they fill the form in 2 seconds like a bot?
   - **Device Agent** — is this IP a VPN/Tor/datacenter? Is the user "physically" in two countries within an hour (impossible travel)?
4. **The Guardrail** — a final safety net that overrides everything if something is *definitely* fraud (like a card-testing attack), forces human review if something looks broken or biased, and writes a permanent audit record.
5. **Your layer again — response + storage + live dashboard push** — sends the decision back to the merchant, saves it to the database, and pushes a live update to the merchant's fraud dashboard over WebSocket.

**Why this matters for you to know:** even though steps 3–4 aren't your code, your API is the entry point and exit point of the *entire system*. You will get asked "what happens after you call the orchestrator?" — you should be able to answer at a high level even though you didn't write that part.

---

## PART 3 — Your Part: What You Actually Built

You are **Member 6 — Backend API Engineer**. Your files:

```
api/routes/analyze.py            ← the main "check this transaction" endpoint
api/routes/transactions.py       ← dashboard: "show me my past transactions"
api/routes/auth.py               ← dashboard login (issues JWT tokens)
api/routes/websocket.py          ← live feed to the dashboard
api/middleware/api_key_auth.py   ← verifies the merchant's API key
api/middleware/rate_limiter.py   ← stops merchants from spamming the API
migrations/001_initial_schema.sql ← the database tables (you own writing this file; schema was co-designed)
```

In one sentence each:

| File | What it does, in one sentence |
|---|---|
| `analyze.py` | The front door — receives a transaction, authenticates it, kicks off the AI pipeline, validates the result, saves it, and replies to the merchant. |
| `transactions.py` | Lets a merchant's dashboard ask "show me my last 50 transactions," filtered and paginated, *only their own data*. |
| `auth.py` | Lets a human (not a merchant's server) log into the dashboard with email/password and get a JWT token. |
| `websocket.py` | Keeps a live, open connection to the dashboard so new fraud decisions appear instantly without refreshing. |
| `api_key_auth.py` | Checks the `Authorization: Bearer ak_live_xxxx` header on every API call and figures out *which merchant* is calling. |
| `rate_limiter.py` | Counts how many requests a merchant made in the last 60 seconds and blocks them with a 429 error if they're over their plan's limit. |

---

## PART 4 — Concepts You MUST Understand Cold

These are the building blocks. If you understand these 12 things, you can answer almost any question thrown at you.

### 1. What is a REST API, really?
It's just a set of URLs that do things when you send HTTP requests to them. `POST /v1/analyze` means "send me data, I'll process it and reply." Think of it like a function call, but over the internet, between two different computers/companies.

### 2. What is FastAPI, and why use it instead of Flask/Django?
FastAPI is a Python web framework built for two things:
- **Speed (async)** — it can handle thousands of requests at once without blocking, because it doesn't sit idle waiting for the database or an external API to respond.
- **Automatic validation** — you describe what a request *should* look like (using Pydantic), and FastAPI rejects anything that doesn't match before your code even runs. It also auto-generates interactive API docs (Swagger UI) for free.

If asked "why not Flask": Flask is synchronous by default (one request blocks the next), and you'd have to manually write validation code. FastAPI was purpose-built for exactly this kind of high-throughput, low-latency, schema-strict API.

### 3. What is `async`/`await`, and why does it matter here?
Your API spends most of its time **waiting** — waiting for the database, waiting for Redis, waiting for the AI agents, waiting for an IP-lookup service. `async`/`await` lets Python say "while I'm waiting on this one thing, go handle a different request instead of just sitting idle." This is *not* the same as multi-threading — it's **concurrency** (juggling many waiting tasks), not **parallelism** (literally doing two things at the exact same nanosecond). This is exactly why you can hit a 200ms SLA even when 3 AI agents and a database write are all happening "at once."

### 4. Two separate auth systems — and why you need BOTH
This is one of the most important things to be able to explain clearly:

- **API Key auth** (`Authorization: Bearer ak_live_xxxx`) — used when the **merchant's server** talks to your API. It's a long-lived, machine-to-machine credential. No login form, no expiry by default, built for server-to-server automation.
- **JWT auth** — used when a **human** (a fraud analyst at the merchant) logs into the *dashboard* with email + password. JWTs expire (24 hours here) because humans log in and out; servers don't.

**Why not just use one system for both?** Because the security model is different. A merchant's *server* needs a credential that doesn't expire mid-transaction and can be rotated/revoked independently. A *human dashboard user* needs a session that expires for security (if they leave their laptop open at a coffee shop, you don't want that session valid forever).

### 5. Why hash the API key instead of storing it directly?
We never store the raw key (`ak_live_xxxx`) in the database — only its SHA-256 hash. SHA-256 is a one-way function: you can turn the key into the hash, but you can't reverse the hash back into the key. So **even if our entire database leaks**, attackers get useless hash values, not usable keys. When a request comes in, we hash the *incoming* key the same way and just compare hashes.

### 6. Multi-tenancy — the most important architectural idea in this whole project
"Tenant" = each merchant (customer business) using AegisFlow. **Every single table has a `merchant_id` column, and every single query is filtered by it.** This is what stops Merchant A from ever seeing Merchant B's transactions. It's not a side feature — it's a structural guarantee baked into every query you write. If asked "how do you prevent data leakage between customers," this is your answer: it's architectural, not just a permission check — the `merchant_id` comes from the authenticated request, never from user input, so there's no way to "ask for" someone else's data even by mistake.

### 7. Rate limiting — sliding window, and why not the simpler version
Each merchant plan has a request limit per minute (Starter: 100/min, Professional: 5,000/min, Enterprise: unlimited). It's implemented with **Redis sorted sets** as a *sliding window*, not a *fixed window*.

- **Fixed window** (simple but flawed): "max 100 requests between :00 and :01." Problem: a merchant could send 100 requests at 11:59:59 and another 100 at 12:00:01 — that's 200 requests in 2 seconds, but each *window* only saw 100. It's gameable at the boundary.
- **Sliding window** (what we built): always looks back exactly 60 seconds from *right now*, no matter when "now" is. You can never burst past the limit by timing requests around a clock boundary.

How it works technically: every request adds a timestamp to a Redis sorted set; before counting, we delete anything older than 60 seconds; then we count what's left. If count > limit → reject with HTTP 429.

### 8. Connection pooling
We don't open a new database connection for every single request — that's slow and expensive. At startup (`main.py`), we create a **pool** of 5–20 reusable connections (`asyncpg.create_pool`). Each request borrows a connection, uses it, and returns it to the pool. This is standard practice for any production API talking to a relational database.

### 9. WebSocket vs. polling — why we used it for the dashboard
A normal HTTP request is "ask once, get one answer, connection closes." A **WebSocket** is a connection that stays open, so the server can *push* new data to the dashboard the instant a new fraud decision happens — no need for the dashboard to keep asking "anything new? anything new?" every few seconds (polling), which wastes resources and adds delay. Real-time fraud dashboards need to show a blocked transaction *immediately*, not 5 seconds later.

### 10. "Fire and forget" / `asyncio.create_task`
After we get a decision, we need to (a) save it to the database and (b) push it to the dashboard via WebSocket. But the merchant is waiting on their checkout page — we don't want to make them wait for those two extra steps. So we kick them off as background tasks (`asyncio.create_task`) and immediately return the response to the merchant. The merchant gets their answer in ~45ms; the DB save and dashboard push happen a few milliseconds after, in the background.

### 11. Idempotency — `ON CONFLICT DO NOTHING`
If the same transaction somehow gets processed twice (network retry, merchant bug), we don't want a duplicate or a crash. The `ON CONFLICT DO NOTHING` SQL clause means: if a row with this transaction ID already exists, just silently skip the insert instead of erroring out.

### 12. SQL injection prevention
Every query uses **parameterized queries** (`$1`, `$2`, etc.) instead of building SQL by string concatenation. This means user-supplied data (like a transaction ID) can never be interpreted as SQL code — the database treats it strictly as a value, not as part of the command. This is the standard, correct way to prevent SQL injection attacks.

---

## PART 5 — Walk Through Your Main Endpoint Like a Story

This is the single most important thing to be able to narrate fluently: **what happens, in order, when `POST /v1/analyze` is called.**

```
1. Merchant's server sends POST /v1/analyze with a Bearer API key + transaction JSON
2. api_key_auth.py: hash the key → look up merchant_id + plan_tier in DB
      → if invalid: return 401 immediately
      → if merchant suspended: return 403 immediately
3. rate_limiter.py: check this merchant's request count in the last 60 sec
      → if over their plan limit: return 429 immediately
4. Pydantic validates the request body shape automatically
      → if malformed: FastAPI returns 422 automatically, my code never even runs
5. analyze.py calls run_fraud_analysis() → this is the orchestrator (Member 2's code)
      → orchestrator fires Transaction/Behavioral/Device agents in parallel
      → returns a FraudDecision object
6. analyze.py calls validate_decision() → this is the guardrail (Member 8's code)
      → can override the decision (e.g. force block, force review)
7. I kick off TWO background tasks (don't block the response):
      a. save the decision to the fraud_decisions table
      b. push the decision to the merchant's WebSocket dashboard feed
8. I return the response to the merchant: decision, score, flags, processing time
```

If you can say this out loud in under 90 seconds without reading it, you're in good shape.

---

## PART 6 — Likely Questions and How to Answer Them

Organize your nerves around this: **a professor grading a group project almost always tests whether you understand your own slice deeply, not whether you memorized your teammates' code.** Expect 70% of questions to be about YOUR part.

### Auth & Security
**Q: Why two different auth systems instead of one?**
> API keys are for machine-to-machine (merchant server → us), long-lived, no session concept. JWT is for human dashboard logins, short-lived (24h), tied to a person's session. Different threat models need different mechanisms.

**Q: What if someone steals an API key?**
> They could make fraudulent calls to our API under that merchant's identity until the merchant notices and revokes it (we'd flip `is_active = FALSE` on that key). This is why keys are scoped per-merchant and revocable individually rather than one global secret.

**Q: Why hash the API key instead of encrypting it?**
> Hashing is one-way — there's no key to "decrypt" with, so even an internal database breach can't expose usable credentials. Encryption would require storing a decryption key somewhere, which becomes its own vulnerability.

**Q: How do you prevent Merchant A from seeing Merchant B's data?**
> Every table has a `merchant_id` foreign key, and `merchant_id` is never taken from user input — it comes only from the authenticated request (decoded from the API key or JWT). Every query in `transactions.py` filters `WHERE merchant_id = $1` using that authenticated value. There's structurally no code path where one merchant's ID can be substituted for another's.

### Performance & Architecture
**Q: Why async instead of normal synchronous Python?**
> Because almost everything we do is I/O-bound — waiting on the database, Redis, or the AI agents — not CPU-bound. Async lets one process handle many requests concurrently while waiting, instead of one request blocking everyone behind it. This is essential to hit our ~200ms response budget under real load.

**Q: How do you keep response time under 200ms?**
> Three things: (1) the orchestrator runs all three AI agents in parallel instead of one after another, (2) database and WebSocket writes happen as background tasks after the response is already sent, (3) we use connection pooling so we're never paying the cost of opening a new DB connection per request.

**Q: What happens if the database is down when a request comes in?**
> The actual fraud decision can still be returned to the merchant since the analysis itself doesn't strictly require the DB write to succeed first — the persistence happens as a background task wrapped in a try/except, so a DB outage degrades logging, not the merchant-facing decision. (Be honest if asked for more depth: this is a known tradeoff — we prioritize availability of the decision over guaranteed logging.)

**Q: Why rate limit per plan tier instead of one global limit?**
> Different merchants pay for different throughput — a Starter merchant doing a few orders a day shouldn't be able to consume the same server capacity as an Enterprise merchant doing thousands per minute. It's also a monetization lever and an abuse-prevention mechanism.

**Q: Explain the sliding window rate limiter in detail.**
> We store a timestamp per request in a Redis sorted set keyed by merchant ID. On every new request, we first remove all entries older than 60 seconds, add the new timestamp, then count how many remain. If that count exceeds the plan's limit, we reject with 429. This avoids the "burst at the boundary" flaw of fixed windows, where you could send double your limit by timing requests around a clock-minute edge.

### Database
**Q: Why is the `transactions` table partitioned by month?**
> At scale (potentially millions of transactions), querying one giant table gets slow. Partitioning by month means PostgreSQL only has to scan the relevant month's partition for most queries (especially dashboard queries filtered by date), which keeps things fast as data grows. Member 7 automates creating next month's partition automatically.

**Q: What's the difference between the `transactions` table and `fraud_decisions` table?**
> `transactions` stores the raw transaction data that was submitted. `fraud_decisions` stores the *output* of our analysis — score, decision, which agent contributed what, SHAP explanation. They're linked by `transaction_id` and joined together when the dashboard needs the full picture.

**Q: Why is `audit_log` insert-only with no updates or deletes allowed?**
> It's a compliance/legal requirement — if a merchant disputes a decision or there's a chargeback investigation, you need an unmodifiable historical record of exactly what was decided and why. If rows could be edited or deleted, the audit trail would be worthless as evidence.

### WebSocket
**Q: Why WebSocket instead of just having the dashboard refresh every few seconds?**
> Polling wastes server resources (constant repeated requests, mostly answered "nothing new") and introduces a delay equal to the polling interval. WebSocket keeps a connection open so we can push a new decision the instant it happens — true real-time, with far less load on the server overall.

**Q: What happens if a dashboard's WebSocket connection drops?**
> The `WebSocketDisconnect` exception is caught, and that specific connection is removed from our `connections` dict for that merchant, so we stop trying to send to a dead socket. (Honest add-on: this version doesn't auto-reconnect or replay missed messages — that's a realistic improvement we'd add for production.)

### General / "gotcha" questions
**Q: What's the actual difference between a 401, 403, and 429 response?**
> 401 = "I don't know who you are" (invalid/missing API key). 403 = "I know who you are, but you're not allowed" (merchant account suspended). 429 = "You're allowed, but you're calling too fast" (rate limit exceeded).

**Q: If you had more time, what would you improve in your part?**
> Good answer to have ready, shows maturity: "I'd add API key rotation without downtime, WebSocket reconnect-with-replay so the dashboard never misses an event during a disconnect, and per-endpoint rate limiting (right now it's global per merchant, not per endpoint)."

**Q: Did you write the fraud detection logic / AI agents?**
> Be honest and clear about scope: "No — that's Members 2 through 5 and 8. My responsibility is the API surface: authentication, rate limiting, the endpoints themselves, the WebSocket feed, and making sure the database layer for the API is multi-tenant-safe. I call into the orchestrator and the guardrail, but I don't implement the scoring logic itself." This is a completely fine and expected answer in a multi-person team project — don't overclaim work that isn't yours.

---

## PART 7 — How to Structure Your Actual Presentation

A clean order that lets you control the conversation:

1. **30 sec — the pitch** (Part 1 above)
2. **1 min — where you sit in the architecture** (draw or point to the diagram: merchant → your API → orchestrator → guardrail → back to merchant)
3. **2–3 min — walk the request lifecycle as a story** (Part 5), narrating it like you're tracing a single transaction live
4. **2 min — the two auth systems and why both exist** (this is usually the most "impressive" thing to explain well, because it shows you understand *security tradeoffs*, not just code)
5. **1–2 min — rate limiting, sliding window explanation** (this is a great one to explain with the burst-at-boundary example — it sounds technical and shows real understanding)
6. **1 min — multi-tenancy** (one sentence: "every table has merchant_id, every query filters on it, that's the whole isolation guarantee")
7. **Close — what you'd improve given more time** (Part 6, last question)

Keep your tone conversational, not memorized-robotic — examiners can tell the difference instantly, and "I understand this and can explain it in my own words" scores much higher than "I can recite this."

---

## PART 8 — If You Get Asked Something You Genuinely Don't Know

Don't bluff. Say: *"That's part of [Member X]'s component — I know it at a high level [give the one-sentence summary from Part 2], but the implementation detail isn't something I can speak to precisely since I focused on the API layer."* This is a normal, professional, and honest answer in a team project — professors respect it far more than a wrong guess.
