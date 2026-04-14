# Case ID to Session Mapping – Design & Implementation Guide

## 🎯 Objective

Enable seamless handling of business-driven inputs (e.g., `case_id`) within an agent-based conversational system by reliably mapping them to `session_id`.

---

# 🧠 Core Concept

* **case_id** → Business identifier (e.g., payment investigation case)
* **session_id** → Conversational runtime context (agent session)

👉 These should **not be the same**. Instead, maintain a **mapping layer**.

---

# 🏗️ High-Level Architecture

```
User Input (case_id or natural language)
        ↓
Pre-Processor (extract case_id)
        ↓
Case-Session Mapping Service
        ↓
Session Resolution (existing/new)
        ↓
Agent Invocation (with context)
        ↓
Session Events (for UI)
```

---

# ✅ Approach 1: Explicit Case-to-Session Mapping (Recommended)

## Description

Maintain a persistent mapping between `case_id` and `session_id`.

## Benefits

* Clean separation of concerns
* Supports audit, replay, and traceability
* Enables multi-agent support

## Schema Example

```sql
CREATE TABLE case_sessions (
    id UUID PRIMARY KEY,
    case_id VARCHAR(100) NOT NULL,
    agent_name VARCHAR(100) NOT NULL,
    session_id VARCHAR(255) NOT NULL,
    status VARCHAR(50) DEFAULT 'ACTIVE',
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    UNIQUE (case_id, agent_name)
);
```

## Flow

1. Receive request with `case_id`
2. Lookup mapping
3. If exists → reuse session
4. Else → create new session + store mapping

---

# ✅ Approach 2: API Design Options

## Option A: Case-Aware Endpoint

```
POST /agents/{agent_name}/cases/{case_id}/runs
```

### Pros

* Clean REST design
* Explicit context

### Cons

* Less flexible for generic UI

---

## Option B: Enriched Payload (Recommended)

```
POST /agents/{agent_name}/runs
{
  "case_id": "CASE123",
  "input": "Investigate issue"
}
```

### Pros

* Works with generic UI (AG-UI)
* Extensible

---

# ✅ Approach 3: Natural Language Input Handling

## Problem

User may input:

```
"investigate case CASE123"
```

## Solution: Pre-Processor Layer

### Responsibilities

* Extract `case_id`
* Clean user intent

---

## Implementation Options

### Option A: Regex-Based Extraction

```python
import re

def extract_case_id(text):
    match = re.search(r'case\s+([A-Z0-9\-]+)', text, re.IGNORECASE)
    return match.group(1) if match else None
```

### Option B: LLM-Based Extraction

Prompt:

```
Extract case_id from the input.
Return JSON:
{ "case_id": "..." }
```

---

## Output Transformation

Input:

```
"investigate case CASE123"
```

Output:

```json
{
  "case_id": "CASE123",
  "input": "investigate"
}
```

---

# ✅ Approach 4: Session Lifecycle Strategy

## Option A: One Case = One Session (Recommended)

### Benefits

* Consistent context
* Easy audit
* Ideal for investigations

---

## Option B: One Case = Multiple Sessions

### Use Cases

* Multi-agent workflows
* Parallel investigations

### Example

```json
{
  "case_id": "CASE123",
  "sessions": ["S1", "S2"]
}
```

---

# ✅ Approach 5: Context Injection

Always pass `case_id` into agent context:

```json
{
  "context": {
    "case_id": "CASE123"
  }
}
```

### Benefits

* Tools can access case data
* Agent reasoning remains consistent

---

# ✅ Approach 6: Hybrid Processing (Best for Enterprise)

## Backend Handles:

* case_id extraction
* session resolution
* mapping persistence

## Agent Handles:

* reasoning
* workflow execution
* tool usage

---

# ⚙️ Example Flow

```
User: "investigate case CASE123"
        ↓
Pre-Processor → extract case_id
        ↓
Resolve session via mapping
        ↓
Invoke agent with:
    session_id + cleaned input + context
        ↓
Agent processes request
```

---

# 🖥️ UI (AG-UI) Integration

## UI Uses:

```json
{
  "case_id": "CASE123"
}
```

## Backend:

* resolves session
* calls `getSessionEvents(session_id)`

👉 UI remains generic and stateless.

---

# 🔒 Production Enhancements

## Idempotency

* Unique constraint on `(case_id, agent_name)`

## Session Expiry

* Add `expires_at`
* Auto-create new session if expired

## Status Management

```
ACTIVE | CLOSED | FAILED
```

## Audit Logging

```sql
CREATE TABLE case_audit_logs (
    id UUID PRIMARY KEY,
    case_id VARCHAR(100),
    action VARCHAR(100),
    metadata JSONB,
    created_at TIMESTAMP DEFAULT NOW()
);
```

---

# 🚫 Anti-Patterns

❌ Using `case_id` as `session_id`
❌ Creating new session on every request
❌ Not extracting `case_id` from user input
❌ Storing mapping only in memory

---

# 🏁 Final Recommendation

Start with:

* Case-session mapping table
* Single session per `(case_id, agent)`
* Backend extraction of `case_id`
* Context injection into agent

Then evolve to:

* multi-agent support
* session expiry
* audit logs

---

# 🧩 Mental Model

* **case_id = Jira Ticket**
* **session_id = Chat Thread**

👉 They are linked, not merged.

---

# 📌 Summary

| Concern            | Solution       |
| ------------------ | -------------- |
| Business context   | case_id        |
| Conversation state | session_id     |
| Mapping            | Database table |
| Natural input      | Pre-processor  |
| Execution          | Agent          |
| UI                 | case_id-driven |

---

**This design ensures scalability, traceability, and clean separation between business and conversational layers.**


# Agent Platform – Storage and Timeline Architecture Strategy

## 1. Purpose

This document defines the target-state architecture for storage and timeline rendering within the Agent Platform built using Google ADK.

The objective is to:

* Improve performance and scalability
* Decouple platform dependencies from ADK internal storage
* Enable a consistent, reusable, and audit-compliant timeline model across use cases

---

## 2. Current State

### 2.1 Data Model

The current implementation relies on:

* **ADK-managed tables**

  * sessions
  * events

* **Platform-managed tables**

  * correlation_sessions
    (maps investigation_id / correlation_id → session_id)

---

### 2.2 UI Interaction Model

#### Session Summary View

* UI retrieves:

  * correlation_sessions
  * sessions
* Displays:

  * List of sessions associated with an investigation (correlation_id)

---

#### Timeline View

* On user selection:

  * System queries events table using session_id
* Displays:

  * Full execution timeline derived directly from ADK events

---

### 2.3 Key Limitations

1. **Performance Constraints**

   * Events table contains large payloads
   * Query latency increases with data volume

2. **Storage Inefficiency**

   * Full payloads (tool outputs, LLM responses) stored in PostgreSQL

3. **Tight Coupling**

   * UI and platform logic directly dependent on ADK schema

4. **Scalability Risks**

   * Multi-tenant adoption will significantly increase load and storage footprint

---

## 3. Target State Architecture

### 3.1 Design Principles

* ADK will be treated as an **execution engine only**
* Platform will maintain its own **derived, query-optimized data model**
* Heavy payloads will be externalized to object storage
* UI will consume only platform-managed data structures

---

## 4. Proposed Data Model

### 4.1 Investigation Timeline Table

A new table will be introduced:

`investigation_timeline`

#### Purpose:

* Provide a lightweight, query-efficient representation of execution events
* Serve as the primary data source for UI timeline rendering

---

### 4.2 Schema Definition

```sql
CREATE TABLE investigation_timeline (
    id BIGSERIAL PRIMARY KEY,
    tenant_id VARCHAR(50),
    investigation_id VARCHAR(100),
    session_id VARCHAR(100),

    event_id VARCHAR(100),

    event_type VARCHAR(50),
    event_subtype VARCHAR(50),
    status VARCHAR(20),

    summary TEXT,
    s3_payload_path TEXT,

    created_at TIMESTAMP DEFAULT NOW()
);
```

---

## 5. Data Flow Architecture

### 5.1 Processing Flow

ADK events will not be consumed directly by UI.

Instead, a transformation layer will be introduced:

```plaintext
ADK (sessions, events)
        ↓
Timeline Processor
        ↓
PostgreSQL (investigation_timeline)
        ↓
S3 (payload storage)
```

---

### 5.2 Payload Management

* Large payloads (tool outputs, detailed logs) will be stored in S3
* PostgreSQL will store:

  * summary
  * metadata
  * S3 reference path

---

## 6. UI Interaction Model (Target State)

### 6.1 Session Summary View

No immediate change:

* Continue using correlation_sessions + sessions

---

### 6.2 Timeline View

Replace direct dependency on events table:

* UI will query:

  * investigation_timeline

This ensures:

* Reduced latency
* Consistent structure across agents

---

### 6.3 Detailed Payload Access

* Full payloads will be retrieved on demand from S3
* Avoids loading large data during timeline rendering

---

## 7. ADK Data Retention Strategy

ADK tables will be treated as transient operational storage.

### Retention Policy

| Table    | Retention Period |
| -------- | ---------------- |
| sessions | 7–14 days        |
| events   | 7–30 days        |

---

### Cleanup Approach

* Scheduled batch deletion or partition-based cleanup
* Ensures controlled database growth

---

## 8. Implementation Strategy

### Phase 1

* Introduce investigation_timeline table

### Phase 2

* Implement timeline processor (polling-based)

### Phase 3

* Backfill historical data

### Phase 4

* Migrate UI to timeline table

### Phase 5

* Introduce S3 payload offloading

### Phase 6

* Enable ADK table cleanup

---

## 9. Benefits

* Improved query performance and UI responsiveness
* Reduced database storage footprint
* Decoupling from ADK internal schema
* Scalable multi-tenant architecture
* Enhanced audit and observability capabilities

---

## 10. Architectural Decision: Event Processing Mechanism

### Options Evaluated

#### Option 1: Inline Hooks (Synchronous Processing)

* Transform events during agent execution

#### Option 2: Background Timeline Processor (Asynchronous Polling)

---

### Recommended Approach: Background Timeline Processor

#### Rationale

* **Loose coupling from ADK internals**
* **Lower operational risk**
* **No impact on agent execution latency**
* **Easier rollout and rollback**
* **Supports reprocessing and backfill**

---

### When to Consider Hooks (Future State)

* Real-time streaming UI is required
* Strong consistency is mandatory
* Event latency must be near-zero

---

## 11. Conclusion

The proposed architecture introduces a clear separation between:

| Layer      | Responsibility                              |
| ---------- | ------------------------------------------- |
| ADK        | Execution (transient)                       |
| PostgreSQL | Queryable timeline (source of truth for UI) |
| S3         | Payload and archival storage                |

This approach ensures scalability, maintainability, and alignment with enterprise-grade platform standards.

