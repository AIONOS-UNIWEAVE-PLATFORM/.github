# Analytics & User Management — Service Explainer

**Repo**: `UNIWEAVE-AIONOS/analytics-user-mgmt`
**Tech stack**: Java 17, Spring Boot 3.5.3, MongoDB, Kafka, Azure Blob Storage
**Originally built by**: Whilter (partner engineering team)
**Purpose**: This is the backend that powers UniWeave's analytics dashboard, user login system, and the pipeline that processes call data after every voice AI conversation ends.

This document explains every service in the repo so a new engineer can understand what each piece does and how they connect.

---

## The big picture

This repo has **8 modules**: 7 running services + 1 shared code library. Here they are:

**The audit pipeline** (this is the part that processes call data):
```
AUDIT-PUBLISHER → Kafka queue → AUDIT-LISTENER → MongoDB → AUDIT-ANALYTICS
```

**Everything else**:
- **API-GATEWAY** — front door, routes all HTTP traffic to the right service
- **USER-MANAGEMENT** — handles login, users, organizations
- **EXTERNAL-API** — stores airline data (flights, PNR, loyalty)
- **TOOL-SERVICE** — calls external APIs on behalf of the voice bot
- **VOICE-BOT-SPECIFICATIONS** — shared code library (not a running service)

Every running service has its own port. The API Gateway sits in front and routes traffic. Kafka and MongoDB are infrastructure — they're not part of this repo, but the services depend on them.

---

## Service-by-service breakdown

### 1. voice-bot-specifications (shared library)

**What it is**: A code library, NOT a running service. It contains all the shared data shapes (models, enums, DTOs) that every other service uses. Think of it as the "dictionary" that all services agree on.

**Port**: None (it's a library, not a server)

**What's inside**:
- **AuditStore** — the main data shape for a call record. Every call that happens gets stored as one AuditStore object. It contains: transcript, call duration, sentiment, intents, whether a human took over, model performance metrics, and more.
- **User, Organization, Role, Permission** — data shapes for users and access control.
- **Flight, PNR, LoyaltyProgram** — airline-specific data shapes.
- **Enums** — predefined lists like CallStatus (ACTIVE, COMPLETED, FAILED), EventType (CREATE, UPDATE, DELETE), Resolution (RESOLVED, UNRESOLVED), TransferReason, etc.
- **Request/Response DTOs** — the exact shape of data that goes into and comes out of each API.

**Why it matters**: If you change a field in this library, it affects every service. Build this first before building anything else.

---

### 2. audit-publisher (receives call data)

**What it is**: The entry point for all post-call data. When a voice AI call ends, the voice service sends the call data HERE.

**Port**: 7000
**Base path**: `/api/audit-publisher`

**How it works**:
1. Voice AI service finishes a call
2. It sends a POST request to `/api/audit-publisher/publish` with the call data
3. audit-publisher validates the data
4. It puts the data onto a Kafka queue (like dropping a letter in a mailbox)
5. Done. It does NOT save to the database directly.

**The one endpoint that matters**:

```
POST /api/audit-publisher/publish

Body: AuditStoreRequest (JSON)
```

**What goes in the request body** (AuditStoreRequest):

| Field | Type | Required? | What it means |
|---|---|---|---|
| orgId | string | Yes | Which organization this call belongs to |
| auditCategory | string | Yes | Category label (e.g., "voicebot_auth_key") |
| eventType | enum | Yes | What happened (CREATE, UPDATE, etc.) |
| sentBy | string | Yes | Name of the service sending this data |
| transcript | list | No | The full conversation, turn by turn |
| modelMetrics | object | No | Performance numbers (latency, tokens used, STT/TTS duration) |
| intents | map | No | What the caller wanted (flight search, PNR check, FAQ, etc.) |
| metaMap | map | No | Free-form metadata (caller number, start time, end time, room name) |
| humanTransfer | boolean | No | Did a human agent take over? |
| transferReason | enum | No | Why the call was transferred |
| status | enum | No | Call status (ACTIVE, COMPLETED, FAILED) |
| resolution | enum | No | Was the issue resolved? (RESOLVED, UNRESOLVED) |
| callDuration | long | No | How long the call lasted (seconds) |
| createdBy | string | No | Who created this record |
| description | string | No | Human-readable summary |

**Kafka topic**: `audit-queue-voicebot-dev`
**Kafka server**: `43.204.24.97:9092`
**Kafka tuning**: Messages are batched before sending (linger.ms=50, batch.size=16384 bytes). This means messages may be delayed up to 50ms to allow batching.

**Health check**: `GET /ping` returns "Pong from Audit Publisher"

---

### 3. audit-listener (processes and saves call data)

**What it is**: Sits and waits for messages on the Kafka queue. When a new call record arrives, it picks it up, adds some extra information, and saves it to MongoDB.

**Port**: 7001
**Base path**: `/api/audit-listener`

**How it works**:
1. Listens to Kafka topic `audit-queue-voicebot-dev`
2. When a message arrives, it deserializes it back into an AuditStore object
3. It fills in missing defaults (empty transcript if none provided, etc.)
4. It calculates call duration from start_time and end_time in the metadata
5. It sets resolution: RESOLVED if no human transfer, UNRESOLVED if there was one
6. It saves the record to MongoDB

**Database**: MongoDB
- Database name: `dev-audit` (configurable)
- Collection: `audit_store`
- Indexed fields: `createdDate`, `auditCategory`, `orgId`

**Important note**: There is commented-out code for calling an AI service to analyze sentiment and intents. This was planned but not activated. The fields exist in the data model but are not automatically filled by this service.

**Kafka consumer settings**: The listener allows up to 10 minutes per message before Kafka considers it dead (`max.poll.interval.ms=600000`). Session timeout is 45 seconds. Auto-offset reset is `latest` — new consumers skip old messages. Auto-commit is off — offsets are managed manually.

**Health check**: `GET /ping` returns "Pong from Audit Listener"

---

### 4. audit-analytics (dashboard query APIs)

**What it is**: The read-only analytics service. It does NOT write data — it only reads from MongoDB and returns analytics for dashboards. This is what powers the IntelliPulse/Pulse dashboard screens.

**Port**: 7002
**Base path**: `/api/audit-analytics`

**All the analytics endpoints**:

| Endpoint | What it returns |
|---|---|
| `GET /analytics/ai-containment` | How many calls the AI handled without needing a human (%) |
| `GET /analytics/escalation-rate` | How many calls got transferred to a human (%) |
| `GET /analytics/sentiment-heatmap` | Sentiment by hour and day (grid view) |
| `GET /analytics/ai-handling-time` | Average time the AI spent per call |
| `GET /analytics/csat-score` | Customer satisfaction scores over time |
| `GET /analytics/csat-daily` | Daily CSAT scores |
| `GET /analytics/escalation-reason` | Why calls got escalated (with percentages) |
| `GET /analytics/automated-insights` | Auto-generated summaries (top FAQ, top route, top escalation reason) |
| `GET /analytics/intent-average-score` | Average sentiment per intent type |
| `GET /analytics/sentiment-distribution` | Sentiment breakdown (positive/negative/neutral %) |
| `GET /analytics/intent-frequency` | How often each intent was detected |
| `GET /analytics/daily-calls` | Number of calls per day |
| `GET /analytics/call-duration` | Average call length |
| `GET /analytics/automation-rate` | AI-handled vs human-handled ratio |
| `GET /analytics/hourly-call-volume` | Call volume by hour of day |
| `GET /analytics/entity-search-flight` | Most searched flight routes |
| `GET /analytics/entity-faqs` | Most asked FAQ categories |

**Audit record endpoints**:

| Endpoint | What it returns |
|---|---|
| `GET /audit/?category=X&pageNumber=0&pageSize=10` | Paginated list of call records |
| `GET /audit/{id}` | One specific call record by ID |
| `GET /audit/room/{id}` | Call record by room name |
| `DELETE /audit/{id}` | Delete a call record |

**Audio playback**: `GET /presigned-urls/s3?analyticsId=X` — downloads the call audio file from Azure Blob Storage (link expires in 20 minutes).

**Common query parameters**: `timeRange`, `responseFormat`, `startDate`, `endDate`

**Health check**: `GET /ping` returns "Pong from Audit Analytics"

---

### 5. api-gateway (front door)

**Port**: 8080

**Routing rules**:

| URL prefix | Target service | Env var to override |
|---|---|---|
| `/api/tool-service/` | tool-service | `INDIGO_TOOL_SERVICE_URL` |
| `/api/user/` | user-management | `USER_MANAGEMENT_URL` |
| `/api/audit-publisher/` | audit-publisher | `AUDIT_PUBLISHER_URL` |
| `/api/audit-analytics/` | audit-analytics | `AUDIT_ANALYTICS_URL` |

**Important**: The default ports in the gateway config do NOT match the ports the services actually listen on. In deployment, the environment variables are set to point to the correct URLs. If running locally, set the env vars or change the gateway config.

---

### 6. user-management (login and users)

**Port**: 8089
**Base path**: `/api/user`

**Key endpoints**:

| Area | Endpoint | What it does |
|---|---|---|
| Login | `POST /auth/login` | Username + password login, returns JWT token |
| Login | `POST /auth/login/google` | Google OAuth login |
| Logout | `POST /auth/logout` | Logs out from one device |
| Token | `POST /auth/refresh` | Get a new access token using refresh token |
| Password | `POST /auth/forgot-password` | Sends password reset email |
| Users | `GET /user/current` | Get the currently logged-in user |
| Orgs | `POST /organization/` | Create an organization |

**Auth system**: JWT tokens (access token + refresh token), RSA key pair for signing, multi-tenancy (every user belongs to an orgId), Google OAuth2 supported.

**Admin bootstrap**: Spring profile `admin-bootstrap` — run on port 9089 to create the initial super-admin account. Use for first-time setup of a new deployment.

---

### 7. external-api (airline data)

**Port**: 8085
**Base path**: `/api/external`

Stores airline-specific data — flights, passenger records (PNR), loyalty programs. For non-airline use cases (like Chalet Hotels), this service is not needed.

---

### 8. tool-service (tool caller)

**Port**: 8084
**Base path**: `/api/tool-service`

Smart middleman. When the voice bot needs to call an external API during a conversation, the request goes through tool-service. It reads `tools-config.yaml` to know which API to call.

**Configured tools**:

| Tool name | What it calls | Method |
|---|---|---|
| `getLoyaltyStatus` | external-api `/loyalty-programs` | GET |
| `getPnrStatus` | external-api `/pnr/search` | GET |
| `flightSearch` | external-api `/flights/search` | GET |
| `ragQuery` | RAG service `/rag/query` | POST |
| `sendWhatsappMsg` | WhatsApp backend `/backend/send-whatsapp-msg` | POST |

**Security**: Every request needs an `x-api-key` header. Set via `SECRET_KEY` env var.

**Note for IntelliRAG (#98)**: The `ragQuery` tool already exists in tool-service. The plumbing for RAG queries is already built — check that the RAG endpoint URL and API key in `tools-config.yaml` are correct. **The default API key is hardcoded** — replace with a proper env var before production.

---

## Port Summary

| Service | Port | Purpose |
|---|---|---|
| api-gateway | 8080 | Front door, routes all traffic |
| audit-publisher | 7000 | Receives post-call data, puts on Kafka |
| audit-listener | 7001 | Reads from Kafka, saves to MongoDB |
| audit-analytics | 7002 | Dashboard query APIs |
| tool-service | 8084 | Calls external APIs for the bot |
| external-api | 8085 | Airline data (flights, PNR, loyalty) |
| user-management | 8089 | Login, users, organizations |

---

## For IntelliPulse Integration (#96)

If you are building the post-call webhook from UniWeave Voice AI to this system:

1. **Send a POST request** to `/api/audit-publisher/publish` when a call ends
2. **Include at minimum**: orgId, auditCategory, eventType, sentBy
3. **Include ideally**: transcript (full conversation), modelMetrics (performance data), intents, metaMap (with start_time, end_time, room_name, mobile_number), humanTransfer flag
4. **The rest happens automatically**: Kafka delivers it to the listener, listener saves to MongoDB, analytics service can query it

You do NOT need to talk to audit-listener or audit-analytics directly. Just send data to audit-publisher and the pipeline handles the rest.

---

## Known Gotchas

**1. The two audit services use different default database names**

audit-listener saves to `dev-audit`. audit-analytics reads from `audit-analytics`. These are DIFFERENT databases. If you deploy with default settings, the dashboard will return nothing. Fix: set `SPRING_DATA_MONGODB_DATABASE` to the same value on BOTH services.

**2. Kafka publish is fire-and-forget — data can be lost silently**

audit-publisher returns 200 OK immediately, before Kafka confirms receipt. If Kafka is down, the error is logged but the caller still gets 200. No DLQ, no retry. For guaranteed delivery (#96), add acknowledgment logic or a DLQ.

**3. There's a dormant AI enrichment feature in audit-listener**

Commented-out Feign client (ToolClient) that would call an external API to auto-analyze intent and sentiment for every call. Relevant to #97 (Pulse enrichment) — the plumbing for auto-enrichment is already built. To activate: uncomment the ToolClient call and configure `TOOLS_API_URL`.

**4. user-management also sends data through the audit pipeline**

User login/create/update events are also published through the same Kafka pipeline. The `audit_store` collection contains both voice call records AND user management events — filter by `auditCategory` to tell them apart.

**5. The audio download class is misnamed**

`S3FileService` in audit-analytics actually uses Azure Blob Storage SDK (not AWS S3). Audio files are stored as `{roomId}.ogg` in an Azure Blob container.

---

## Infrastructure

| Component | Details |
|---|---|
| Language | Java 17 |
| Framework | Spring Boot 3.5.3 |
| Database | MongoDB (multiple databases: dev-audit, voicebotdb, external-api) |
| Message queue | Apache Kafka (topic: audit-queue-voicebot-dev) |
| Audio storage | Azure Blob Storage (call recordings) |
| Auth | JWT with RSA key pair + optional Google OAuth |
| Build tool | Maven (parent POM at root) |
| Build order | voice-bot-specifications first, then everything else |

---

## Related

- [System Architecture](system-architecture.md) — full platform overview
- [IntelliPulse Integration Spec](specs/intellipulse-integration.md)
