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
