# Task Tracker — Serverless AWS Architecture Guide

> This document describes how the Task Tracker would be re-architected on AWS
> using fully managed, serverless services. No code is written here — this is
> the design and reasoning document to read before implementation begins.

---

## Table of Contents

1. [Why Go Serverless?](#1-why-go-serverless)
2. [High-Level Architecture](#2-high-level-architecture)
3. [Component-by-Component Breakdown](#3-component-by-component-breakdown)
   - 3.1 Amazon S3 — Static Frontend Hosting
   - 3.2 Amazon CloudFront — CDN and Single Entry Point
   - 3.3 Amazon API Gateway — HTTP API
   - 3.4 AWS Lambda — Five Function Handlers
   - 3.5 Amazon DynamoDB — Task Database
   - 3.6 AWS IAM — Permissions and Security
   - 3.7 Amazon CloudWatch — Logs and Monitoring
   - 3.8 AWS X-Ray — Distributed Tracing
4. [Data Model: SQLite vs DynamoDB](#4-data-model-sqlite-vs-dynamodb)
5. [End-to-End Request Flows](#5-end-to-end-request-flows)
6. [How Security Issues Are Resolved](#6-how-security-issues-are-resolved)
7. [Cost Breakdown](#7-cost-breakdown)
8. [What Changes vs. the Current App](#8-what-changes-vs-the-current-app)
9. [Implementation Roadmap](#9-implementation-roadmap)
10. [Glossary](#10-glossary)

---

## 1. Why Go Serverless?

The current Task Tracker runs on a single machine. To use it, that machine must
be switched on and `python app.py` must be running. If the machine goes offline,
the app is gone. If two people try to use it at once, they both write to the same
`tasks.db` file, which can corrupt data.

**Serverless** means you stop managing any server at all. You write your
application logic, upload it to AWS, and AWS handles:

- Starting and stopping machines when requests come in and go idle
- Scaling up automatically if 1 person or 1,000 people use the app simultaneously
- Patching the operating system and runtime
- Keeping the app available across multiple data centers

You pay only for what you actually use — down to the millisecond of compute time
and the number of database reads/writes. At low volume (personal or small-team
use), the AWS free tier covers almost all of it at **$0/month**.

---

## 2. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              Internet                                   │
└───────────────────────────────┬─────────────────────────────────────────┘
                                │
                    ┌───────────▼────────────┐
                    │     Amazon CloudFront   │  CDN · DDoS protection
                    │   (single entry point)  │  HTTPS termination
                    └──────┬──────────┬───────┘
                           │          │
              /index.html  │          │  /api/*
              (cached)     │          │  (forwarded, not cached)
                           │          │
              ┌────────────▼──┐  ┌────▼──────────────────────┐
              │  Amazon S3    │  │  API Gateway (HTTP API)    │
              │  (static site)│  │  5 routes · JWT auth-ready │
              └───────────────┘  └────┬───────────────────────┘
                                      │  Lambda proxy integration
                    ┌─────────────────┼──────────────────────┐
                    │                 │                        │
           ┌────────▼──────┐  ┌──────▼──────┐  ┌────────────▼──────┐
           │ ListTasks  Fn │  │ CreateTask  │  │  GetTask      Fn  │
           │ GET /tasks    │  │ POST /tasks │  │  GET /tasks/{id}  │
           └────────┬──────┘  └──────┬──────┘  └────────────┬──────┘
                    │                 │                        │
                    ├─────────────────┼────────────────────────┤
                    │                 │                        │
           ┌────────▼──────┐  ┌──────▼──────┐                │
           │ UpdateTask Fn │  │ DeleteTask  │                │
           │ PUT /tasks/{id}│  │DELETE /{id}│                │
           └────────┬──────┘  └──────┬──────┘                │
                    │                 │                        │
                    └─────────────────┼────────────────────────┘
                                      │  IAM role (least privilege)
                           ┌──────────▼──────────┐
                           │   Amazon DynamoDB    │
                           │   Table: Tasks       │
                           │                      │
                           │  PK: id (UUID)       │
                           │  GSI-1: StatusIndex  │
                           │  GSI-2: PriorityIndex│
                           └──────────────────────┘

┌─────────────────── Observability (cross-cutting) ────────────────────────┐
│  CloudWatch Logs (all Lambdas)  ·  CloudWatch Alarms  ·  X-Ray Tracing  │
└──────────────────────────────────────────────────────────────────────────┘
```

**Reading the diagram:**

- The browser always talks to **CloudFront** first — never directly to S3 or API Gateway.
- CloudFront decides where to route the request based on the URL path.
  - Requests to `/` or `/index.html` go to **S3** (the HTML page).
  - Requests to `/api/*` go to **API Gateway**, which routes to the right **Lambda**.
- Each Lambda reads from or writes to **DynamoDB**.
- Every Lambda automatically writes logs to **CloudWatch**.

---

## 3. Component-by-Component Breakdown

---

### 3.1 Amazon S3 — Static Frontend Hosting

**What it is:** S3 (Simple Storage Service) is AWS's object storage service. You
upload files to it the same way you save files to a folder. It can also serve
those files over HTTP — making it a perfect host for the `index.html` frontend.

**What it replaces:** The current app serves `index.html` using Flask's
`send_from_directory`. That means the Python app must be running just to deliver
a static HTML file. S3 serves the same file without any Python process at all.

**How it works:**
1. During deployment, the build tool (CDK or SAM) uploads `static/index.html`
   to an S3 bucket.
2. The bucket is configured as a **static website origin** — it serves files
   over HTTPS.
3. The bucket is **not** publicly accessible on its own. Only CloudFront has
   permission to read from it (via an **Origin Access Control** policy). This
   prevents people from bypassing CloudFront and hitting the bucket directly.

**Key properties:**
- Stores up to 5 TB for free on the AWS free tier (first 5 GB).
- 99.999999999% (11 nines) durability — your files are automatically replicated
  across multiple physical locations within a region.
- No server to manage. Files are just stored.

---

### 3.2 Amazon CloudFront — CDN and Single Entry Point

**What it is:** CloudFront is AWS's Content Delivery Network (CDN). It has
**edge locations** in cities around the world (400+). When a user in London
requests `index.html`, CloudFront serves it from a London edge location instead
of making the user wait for a round-trip to, say, us-east-1.

**What it replaces:** Nothing directly in the current app — the current app has
no CDN. This is a new capability.

**Why it matters:**
1. **Performance** — static assets are cached at the edge closest to the user.
   The browser loads the page faster.
2. **Single domain** — both the frontend (`/`) and the API (`/api/*`) live under
   the same domain (e.g., `tasks.example.com`). No CORS configuration needed
   because the browser sees everything as the same origin.
3. **Security** — CloudFront includes AWS Shield Standard (free DDoS protection)
   automatically. You can also attach AWS WAF (Web Application Firewall) rules
   to block malicious traffic before it reaches the backend.
4. **HTTPS everywhere** — CloudFront terminates HTTPS at the edge using a free
   certificate from AWS Certificate Manager (ACM).

**How routing works:**

CloudFront uses **behaviors** (routing rules based on URL path) to decide which
origin to send a request to:

| URL pattern | Behavior | Sent to |
|---|---|---|
| `/` or `/index.html` | Cached, TTL 1 day | S3 bucket |
| `/api/*` | Not cached, all HTTP methods forwarded | API Gateway |

**Example:** A user visits `https://tasks.example.com/api/tasks`. CloudFront
sees the `/api/` prefix, matches the second behavior, and forwards the request
to API Gateway. The response is returned to the user. CloudFront does not cache
API responses (they change too often and contain user data).

---

### 3.3 Amazon API Gateway — HTTP API

**What it is:** API Gateway is a managed service that accepts HTTP requests,
routes them to the right backend (a Lambda function in this case), and returns
the response. You define the routes (e.g., `GET /tasks`, `POST /tasks`), and
API Gateway handles the rest — TLS termination, request/response mapping,
throttling, and logging.

**Which type — REST API or HTTP API?**

AWS offers two API Gateway flavors. For this app, **HTTP API (v2)** is chosen
because:

- It costs ~70% less per request than REST API (v1).
- It has lower latency.
- It supports native **JWT authorization** — ready to add login via Cognito or
  Auth0 later without redesigning the API.
- The features NOT available on HTTP API (usage plans, response caching, AWS WAF
  at the API layer) are not needed here because CloudFront covers them upstream.

**The 5 routes:**

| Method | Path | Lambda function |
|---|---|---|
| GET | /api/tasks | ListTasksFn |
| POST | /api/tasks | CreateTaskFn |
| GET | /api/tasks/{id} | GetTaskFn |
| PUT | /api/tasks/{id} | UpdateTaskFn |
| DELETE | /api/tasks/{id} | DeleteTaskFn |

**Throttling:** API Gateway enforces a default of 10,000 requests per second
per account. You can set lower limits per stage or route to prevent abuse.

**What it replaces:** The Flask route handlers in `routes.py`. Each `@api.get`,
`@api.post`, etc. decorator maps to one API Gateway route.

---

### 3.4 AWS Lambda — Five Function Handlers

**What it is:** Lambda lets you run a function (a piece of code) in response to
an event — without provisioning any server. AWS starts a compute environment,
runs your function, returns the result, and shuts the environment down. You are
billed for the duration your code runs, rounded to the nearest millisecond.

**Cold starts:** The first time a function is called (or after a period of
inactivity), AWS must start a new execution environment. This "cold start" adds
a small delay (typically 100–500 ms for Python). Subsequent calls to a warm
environment are fast (< 10 ms overhead). For a task tracker, this is acceptable.

**What it replaces:** The route handler functions in `routes.py` and the data
access functions in `db.py`. Each Lambda replaces one route + its corresponding
db.py calls.

**The five functions:**

#### `ListTasksFn`
- **Trigger:** `GET /api/tasks`
- **Input:** Optional query parameters `?status=open` and/or `?priority=high`
- **Logic:**
  1. If `status` filter provided → query the `StatusIndex` GSI in DynamoDB.
  2. If `priority` filter provided → query the `PriorityIndex` GSI.
  3. If no filter → scan the entire table.
  4. Validate filter values are within allowed enums, return 400 if not.
- **Output:** JSON array of task objects, HTTP 200.

#### `CreateTaskFn`
- **Trigger:** `POST /api/tasks`
- **Input:** JSON body with `title` (required), `description`, `priority`,
  `status`, `due_date` (optional)
- **Logic:**
  1. Validate fields (title non-empty, priority/status enums, due_date format).
  2. Generate a UUID for the new task ID.
  3. Set `created_at` to the current UTC timestamp.
  4. Write item to DynamoDB with `PutItem`.
- **Output:** Full task object, HTTP 201.

#### `GetTaskFn`
- **Trigger:** `GET /api/tasks/{id}`
- **Input:** Path parameter `id` (UUID string)
- **Logic:** Call DynamoDB `GetItem` with the given ID.
- **Output:** Task object (200) or `{"error": "task not found"}` (404).

#### `UpdateTaskFn`
- **Trigger:** `PUT /api/tasks/{id}`
- **Input:** Path parameter `id`, JSON body with any subset of fields
- **Logic:**
  1. Validate the supplied fields.
  2. Build a DynamoDB `UpdateItem` expression for only the fields present in the
     request body — untouched fields are not modified (partial update).
  3. If the item does not exist, return 404.
- **Output:** Updated task object, HTTP 200.

#### `DeleteTaskFn`
- **Trigger:** `DELETE /api/tasks/{id}`
- **Input:** Path parameter `id`
- **Logic:** Call DynamoDB `DeleteItem`. Use a condition expression to detect
  whether the item existed.
- **Output:** `{"deleted": true}` (200) or `{"error": "task not found"}` (404).

**Runtime and tooling:**

- Language: **Python 3.13** (matches the existing codebase)
- Library: **AWS Lambda Powertools for Python** — provides structured JSON
  logging, X-Ray tracing, input validation, and idempotency utilities with
  minimal boilerplate
- Each function is packaged independently with only its dependencies

**Memory and timeout settings (recommended):**

| Function | Memory | Timeout |
|---|---|---|
| ListTasksFn | 256 MB | 10 s |
| CreateTaskFn | 256 MB | 10 s |
| GetTaskFn | 128 MB | 5 s |
| UpdateTaskFn | 256 MB | 10 s |
| DeleteTaskFn | 128 MB | 5 s |

256 MB gives approximately 0.14 vCPU. DynamoDB calls are fast (single-digit ms),
so these timeouts are very conservative.

---

### 3.5 Amazon DynamoDB — Task Database

**What it is:** DynamoDB is AWS's fully managed NoSQL key-value and document
database. Like Lambda, there is no server to manage — it scales capacity
automatically and bills per read/write operation.

**What it replaces:** The `tasks.db` SQLite file and the entire `db.py` module.

**Why not keep SQLite?**
Lambda functions are stateless. Each invocation may run on a different physical
server. There is no shared filesystem — a `tasks.db` file written by one Lambda
invocation is invisible to another. DynamoDB is a network service that all Lambda
invocations connect to, so it acts as the single shared store.

**Why not use RDS (a traditional SQL database)?**
RDS runs a persistent server (even when no requests are coming in), which costs
money 24/7. Amazon Aurora Serverless v2 is an option that scales to near-zero,
but it still has a minimum cost and higher latency than DynamoDB. For a simple
task tracker with no complex joins, DynamoDB is the best fit.

**Table design:**

```
Table name: Tasks

Primary Key
  Partition Key:  id          (String)   — UUID, e.g. "a1b2c3d4-..."

Attributes (all tasks):
  title           String
  description     String
  priority        String      — "low" | "medium" | "high"
  status          String      — "open" | "done"
  due_date        String      — "YYYY-MM-DD" or absent (null)
  created_at      String      — ISO 8601 UTC timestamp

Global Secondary Indexes (GSIs):
  StatusIndex
    Partition Key:  status      (String)
    Sort Key:       created_at  (String)
    → Enables efficient query: "give me all tasks with status=done, newest first"

  PriorityIndex
    Partition Key:  priority    (String)
    Sort Key:       created_at  (String)
    → Enables efficient query: "give me all tasks with priority=high, newest first"
```

**Why UUIDs instead of integer IDs?**
SQLite auto-increments a counter to assign IDs (1, 2, 3, …). That counter lives
in the database file and requires a lock. DynamoDB is distributed across multiple
servers — there is no single counter to increment safely at high speed. UUIDs
(e.g., `a1b2c3d4-e5f6-7890-abcd-ef1234567890`) are generated locally by the
Lambda function before writing, so no coordination is needed.

**Billing mode:** On-demand (pay-per-request). There is no capacity to
pre-provision. DynamoDB scales to zero when idle and scales up instantly.

**Free tier:** 25 GB storage + 25 write capacity units + 25 read capacity units
per month — permanently free, not just for 12 months. A personal task tracker
will never exceed this.

---

### 3.6 AWS IAM — Permissions and Security

**What it is:** IAM (Identity and Access Management) controls who and what can
access AWS resources. Every Lambda function runs under an **execution role** — an
IAM identity that defines exactly which AWS API calls the function is allowed to
make.

**Principle of least privilege:** Each function gets only the permissions it
actually needs. `GetTaskFn` only needs `dynamodb:GetItem` on the Tasks table.
It cannot list, create, or delete tasks. If that function is ever compromised,
the blast radius is limited.

**Example IAM policy for `DeleteTaskFn`:**
```json
{
  "Effect": "Allow",
  "Action": "dynamodb:DeleteItem",
  "Resource": "arn:aws:dynamodb:us-east-1:123456789:table/Tasks"
}
```

**Permission matrix:**

| Function | DynamoDB permissions |
|---|---|
| ListTasksFn | `Query`, `Scan` on table + both GSIs |
| CreateTaskFn | `PutItem` on table |
| GetTaskFn | `GetItem` on table |
| UpdateTaskFn | `UpdateItem`, `GetItem` on table |
| DeleteTaskFn | `DeleteItem` on table |

No function has `dynamodb:*` (wildcard). No function has access to any other AWS
service unless explicitly added.

---

### 3.7 Amazon CloudWatch — Logs and Monitoring

**What it is:** CloudWatch collects logs, metrics, and alarms from AWS services.
Every Lambda invocation automatically writes its output to a CloudWatch Log
Group. No configuration is needed — it happens by default.

**What you can do with it:**

- **View logs:** See every request, error, and stack trace in real time or
  historically. Searchable with CloudWatch Logs Insights (SQL-like query
  language).
- **Set alarms:** "Alert me if more than 5 Lambda errors happen in 1 minute."
  Alarms can send an email via SNS.
- **Dashboard:** Build a visual dashboard showing request counts, error rates,
  and Lambda duration over time.
- **Log retention:** Set logs to expire after 30 days to avoid unbounded storage
  costs.

**What it replaces:** Manually checking terminal output from `python app.py`.
CloudWatch stores every log line permanently (up to the retention period) and is
queryable across all five Lambda functions at once.

---

### 3.8 AWS X-Ray — Distributed Tracing

**What it is:** X-Ray traces a single user request as it flows through multiple
AWS services. For the task tracker, a single `POST /api/tasks` request touches:
CloudFront → API Gateway → Lambda → DynamoDB. X-Ray records a timeline of
each step and shows where time was spent.

**Why it matters:** If a request is slow, X-Ray shows immediately whether the
slowness is in Lambda startup (cold start), in the Lambda function code, or in
the DynamoDB call. Without tracing, you would have to guess.

**How it works:** AWS Lambda Powertools includes a `@tracer` decorator. Adding
`@tracer.capture_lambda_handler` to each function automatically creates an X-Ray
trace segment. Each DynamoDB call adds a sub-segment with its own timing.

---

## 4. Data Model: SQLite vs DynamoDB

### Current SQLite schema

```sql
CREATE TABLE tasks (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    title       TEXT    NOT NULL,
    description TEXT    NOT NULL DEFAULT '',
    priority    TEXT    NOT NULL DEFAULT 'medium',
    status      TEXT    NOT NULL DEFAULT 'open',
    due_date    TEXT,
    created_at  TEXT    NOT NULL
)
```

### DynamoDB equivalent

DynamoDB has no `CREATE TABLE` SQL. Instead you define the table, its key schema,
and its GSIs in CDK or SAM. The "schema" for each item is flexible — DynamoDB
does not enforce that every item has the same attributes. The Lambda functions
enforce the schema in application code (input validation).

**Migration differences:**

| SQLite | DynamoDB | Reason |
|---|---|---|
| `id INTEGER AUTOINCREMENT` | `id String (UUID)` | No shared sequence counter in distributed system |
| `NOT NULL DEFAULT ''` enforced by DB | Enforced by Lambda validation | DynamoDB does not enforce column constraints |
| Filter with `WHERE status = ?` | Query GSI `StatusIndex` | DynamoDB requires a key-based access pattern for efficient filtering |
| Order by `id` | Order by `created_at` on GSIs | Integer IDs are sequential; UUIDs are random, so sort on timestamp instead |

---

## 5. End-to-End Request Flows

### Opening the app

```
1. Browser → HTTPS GET https://tasks.example.com/
2. CloudFront checks edge cache.
   Cache HIT (after first load): serves index.html from edge, done.
   Cache MISS (first load):
3. CloudFront → S3: GET index.html
4. S3 → CloudFront: 200 index.html (HTML/CSS/JS)
5. CloudFront caches it at the edge, returns to browser.
6. Browser parses HTML, runs JS → fetch('/api/tasks')
7. → Step: "Loading all tasks" (see below)
```

### Loading all tasks (GET /api/tasks)

```
1. Browser fetch('/api/tasks')
2. → CloudFront (no cache for /api/*)
3. → API Gateway HTTP API: matches route GET /api/tasks
4. → Invokes ListTasksFn (Lambda)
5.   Lambda handler runs:
       a. Parse query params (status, priority) from event
       b. Validate values (400 if invalid)
       c. DynamoDB Scan or Query (GSI if filter present)
       d. Return [{id, title, ...}, ...] as JSON
6. → API Gateway → CloudFront → Browser
7. Browser renders task cards
Total time: ~20-80 ms (warm Lambda + DynamoDB)
```

### Creating a task (POST /api/tasks)

```
1. User fills form, clicks "Add Task"
2. Browser → POST /api/tasks  body: {"title":"Buy milk","priority":"low"}
3. → CloudFront → API Gateway → CreateTaskFn (Lambda)
4.   Lambda handler runs:
       a. Parse JSON body from event
       b. Validate: title non-empty, priority in [low,medium,high],
          status in [open,done], due_date matches YYYY-MM-DD
       c. If invalid → return {"errors":[...]} HTTP 400 (Lambda returns, done)
       d. Generate UUID: id = str(uuid4())
       e. Set created_at = datetime.now(UTC).isoformat()
       f. DynamoDB PutItem: write all fields
       g. Return full task object
5. → API Gateway → CloudFront → Browser  HTTP 201
6. Browser adds new card to the list
```

### Filtering by status (GET /api/tasks?status=done)

```
1. User changes Status filter to "Done"
2. Browser → GET /api/tasks?status=done
3. → CloudFront → API Gateway → ListTasksFn (Lambda)
4.   Lambda handler runs:
       a. status = "done" (from event.queryStringParameters)
       b. priority = None (not supplied)
       c. Validate "done" is in VALID_STATUSES ✓
       d. DynamoDB Query:
            TableName = "Tasks"
            IndexName = "StatusIndex"
            KeyConditionExpression = "status = :s"
            ExpressionAttributeValues = {":s": "done"}
            ScanIndexForward = False  (newest first via created_at sort key)
       e. Return matching tasks
5. → Browser: re-renders list with only done tasks
```

### Marking a task done (PUT /api/tasks/{id})

```
1. User clicks "Mark Done" on a card
2. Browser → PUT /api/tasks/a1b2c3d4-... body: {"status":"done"}
3. → CloudFront → API Gateway → UpdateTaskFn (Lambda)
4.   Lambda handler runs:
       a. id = "a1b2c3d4-..." (from event.pathParameters)
       b. data = {"status": "done"}
       c. Validate: "done" in VALID_STATUSES ✓
       d. Build DynamoDB UpdateItem:
            Key = {"id": id}
            UpdateExpression = "SET #status = :status"
            ConditionExpression = "attribute_exists(id)"  ← fails if not found
       e. If ConditionCheckFailed → return 404
       f. Fetch updated item, return it
5. → Browser: updates card badge and button text
```

### Deleting a task (DELETE /api/tasks/{id})

```
1. User clicks "Delete"
2. Browser → DELETE /api/tasks/a1b2c3d4-...
3. → CloudFront → API Gateway → DeleteTaskFn (Lambda)
4.   Lambda handler runs:
       a. DynamoDB DeleteItem with ConditionExpression = "attribute_exists(id)"
       b. If ConditionCheckFailed → return {"error": "task not found"} 404
       c. Else → return {"deleted": true} 200
5. → Browser: fades out and removes the card
```

---

## 6. How Security Issues Are Resolved

The security audit of the current app found 8 issues. Here is how each one is
addressed architecturally in the serverless design — most without writing a
single line of application code.

| # | Severity | Current Issue | Serverless Resolution |
|---|---|---|---|
| 1 | Critical | `debug=True` enables Werkzeug RCE | Flask and Werkzeug are gone. Lambda has no interactive debugger. |
| 2 | High | No HTTP security headers | CloudFront Response Headers Policy adds `Content-Security-Policy`, `X-Frame-Options: DENY`, `X-Content-Type-Options: nosniff`, `Referrer-Policy` as a managed config — no code needed. |
| 3 | High | No input length limits | Lambda Powertools `Parser` with Pydantic models enforces `max_length` on every field at the schema level before any business logic runs. API Gateway enforces a 10 MB max payload at the network layer. |
| 4 | Medium | XSS gaps in `escapeHtml` | The CSP header from Finding 2 adds `script-src 'self'` — even if an XSS payload reached the DOM, it cannot load remote scripts. The escapeHtml fix should still be applied in the frontend. |
| 5 | Medium | No rate limiting | API Gateway stage throttling (e.g. 100 req/s per route) + CloudFront rate-based WAF rule (block IPs exceeding N req/5min). |
| 6 | Medium | No `SECRET_KEY` | Not needed. Lambda functions are stateless — no server-side sessions exist. If auth is added, JWT tokens are signed by Cognito/Auth0, not by Flask. |
| 7 | Low | Error details in browser console | Lambda returns structured JSON errors only. No stack trace or file path is ever included in the response body. Detailed tracebacks go to CloudWatch (server-side only). |
| 8 | Low | No authentication | API Gateway HTTP API has a **JWT authorizer** slot ready. Connecting Cognito (AWS's auth service) or Auth0 is a one-line CDK change — no Lambda code changes needed. |

**Additional security improvements the serverless architecture adds automatically:**

- **SQL injection is impossible.** DynamoDB uses a typed API (`PutItem`,
  `GetItem`, `Query`) — there is no SQL string to inject into.
- **No hardcoded secrets.** Database "credentials" are replaced by IAM roles.
  Lambda assumes its execution role automatically via the AWS SDK — no password
  to store, rotate, or leak.
- **Encryption at rest.** DynamoDB encrypts all data at rest by default using
  AWS-managed keys (no extra charge).
- **Encryption in transit.** All connections (browser → CloudFront, CloudFront →
  API Gateway, API Gateway → Lambda, Lambda → DynamoDB) use TLS 1.2+.

---

## 7. Cost Breakdown

Estimates based on **10,000 API requests per month** (a reasonable personal/team
workload):

| Service | Usage | Cost |
|---|---|---|
| **CloudFront** | 10,000 HTTPS requests | Free (1 TB transfer + 10M requests/month free tier) |
| **S3** | 1 file (index.html, ~50 KB) | Free (5 GB storage free tier) |
| **API Gateway (HTTP API)** | 10,000 requests | Free (1M requests/month free tier) |
| **Lambda** | 10,000 invocations × 50 ms avg × 256 MB | Free (1M requests + 400,000 GB-s/month free tier) |
| **DynamoDB** | ~30,000 reads + ~15,000 writes | Free (25 WCU + 25 RCU free tier, permanent) |
| **CloudWatch Logs** | ~50 MB logs/month | Free (5 GB ingestion free tier) |
| **Total** | | **$0.00 / month** |

At **1,000,000 requests per month** (high-traffic team):

| Service | Cost |
|---|---|
| CloudFront | ~$0.85 |
| API Gateway | ~$1.00 |
| Lambda | ~$0.21 |
| DynamoDB (on-demand) | ~$1.50 |
| **Total** | **~$3.56 / month** |

Compare to a t3.micro EC2 instance running 24/7: ~$8.50/month before storage,
bandwidth, or any management overhead.

---

## 8. What Changes vs. the Current App

### Code that is replaced

| Current file | Replaced by |
|---|---|
| `app.py` | CDK/SAM infrastructure definition + CloudFront config |
| `routes.py` | 5 Lambda handler functions |
| `db.py` | AWS SDK (boto3) DynamoDB calls inside each handler |
| `static/index.html` | Same file, uploaded to S3 (one URL change: `/api/` prefix stays the same) |
| `requirements.txt` | `requirements.txt` per Lambda (Flask removed, `aws-lambda-powertools` added) |
| `tasks.db` | DynamoDB table (created by CDK/SAM deploy, not a file) |

### Code that stays the same

- `static/index.html` — the frontend works unchanged. It calls `/api/tasks`
  with the same JSON shapes. CloudFront ensures it reaches the right backend.
- Input validation logic — the same rules (title required, priority enum,
  `YYYY-MM-DD` date format) are reimplemented in the Lambda handlers using
  Powertools Parser/Pydantic.
- HTTP status codes and JSON response shapes — 200, 201, 400, 404 and the same
  `{"error": "..."}` / `{"errors": [...]}` format. The frontend does not change.

### Operational differences

| Current | Serverless |
|---|---|
| `python app.py` to start | `cdk deploy` to deploy (one-time, then auto-runs) |
| Crashes if the machine reboots | Runs forever; AWS manages availability |
| One user at a time (SQLite write lock) | Thousands of concurrent users |
| Logs in terminal only | Logs in CloudWatch, searchable, retained |
| Manual restart on failure | Auto-retry on transient errors |
| Debug with `print()` | Structured JSON logs + X-Ray traces |

---

## 9. Implementation Roadmap

When ready to build, the work splits into these phases:

### Phase 1 — Infrastructure scaffolding (CDK)
- Create CDK project (TypeScript)
- Define the DynamoDB table with its GSIs
- Define 5 Lambda functions with execution roles
- Define HTTP API Gateway with routes pointing to Lambdas
- Define S3 bucket for static assets
- Define CloudFront distribution with two origins and behaviors
- Deploy to AWS (`cdk deploy`) — no application code yet

### Phase 2 — Lambda handlers (Python)
- Implement each of the 5 handlers
- Use Powertools Logger, Tracer, and Parser
- Write unit tests (pytest) for each handler with a mocked DynamoDB client
- Write integration tests against a local DynamoDB (via LocalStack or
  `dynamodb-local`)

### Phase 3 — Frontend update
- Update the API base URL in `index.html` from `http://localhost:5000` to the
  CloudFront domain
- Upload to S3 as part of `cdk deploy`

### Phase 4 — Observability
- Configure CloudWatch log retention (30 days)
- Create a CloudWatch dashboard: request count, error count, Lambda p95 duration
- Set an alarm: "email me if error rate > 1% over 5 minutes"

### Phase 5 — Hardening (optional)
- Attach AWS WAF to CloudFront (rate-based rule, SQL injection rule)
- Add CloudFront Response Headers Policy (security headers)
- Add Cognito JWT authorizer to API Gateway for authentication
- Enable DynamoDB Point-in-Time Recovery (PITR) for backup

---

## 10. Glossary

| Term | Plain-English explanation |
|---|---|
| **Serverless** | You write code; a cloud provider runs it on demand without you managing servers |
| **Lambda function** | A piece of code that runs in response to an event (e.g., an HTTP request) and stops when done |
| **Cold start** | The brief delay (~100–500 ms) when Lambda starts a new execution environment for the first time |
| **DynamoDB** | AWS's managed NoSQL database; stores data as items (like JSON objects) rather than rows in a table |
| **GSI (Global Secondary Index)** | A second index on a DynamoDB table that lets you query by a different key — like adding a `WHERE status = ?` fast path |
| **API Gateway** | A managed service that accepts HTTP requests and routes them to backend services |
| **CloudFront** | AWS's CDN; caches and serves content from edge locations close to users worldwide |
| **IAM** | AWS's permission system; controls what each service or user is allowed to do |
| **CDK** | AWS Cloud Development Kit; lets you define AWS infrastructure as TypeScript/Python code |
| **SAM** | AWS Serverless Application Model; a YAML-based tool for defining and deploying serverless apps |
| **UUID** | A randomly generated unique ID (e.g., `a1b2c3d4-e5f6-7890-abcd-ef1234567890`); used instead of auto-increment integers |
| **JWT** | JSON Web Token; a signed, self-contained token used to authenticate API requests |
| **TLS** | Transport Layer Security; the protocol behind HTTPS that encrypts data in transit |
| **WAF** | Web Application Firewall; inspects HTTP requests and blocks malicious traffic |
| **X-Ray** | AWS's distributed tracing service; shows how long each step of a request takes |
| **Origin Access Control** | An S3 permission that only allows CloudFront (not public internet) to read the bucket |
| **Powertools** | AWS Lambda Powertools for Python; a library that adds structured logging, tracing, and validation to Lambda functions with minimal code |
