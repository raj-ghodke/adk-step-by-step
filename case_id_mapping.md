# Agent Platform – Storage and Session Management Strategy (S3-Based Model)

## 1. Purpose

This document defines the revised storage and session management strategy for the Agent Platform built using Google ADK.

The objective is to:

* Avoid storing business data within the platform (compliance requirement)
* Delegate full session and event storage to agent-owning teams
* Maintain platform ownership of execution metadata only
* Establish a scalable and decoupled architecture for multi-tenant usage

---

## 2. Design Principles

* Google ADK is used strictly as an **execution engine**
* Platform does **not own business or session payload data**
* All session and event data is stored in **team-owned S3 buckets**
* Platform maintains only **metadata required for orchestration and UI**
* UI interacts with S3 **only via platform APIs**

---

## 3. Current State (For Reference)

### Data Storage

* ADK tables:

  * sessions
  * events
* Platform table:

  * correlation_sessions (correlation_id → session_id)

### UI Behavior

* List view:

  * correlation_sessions + sessions
* Detail view:

  * events table queried using session_id

### Limitations

* Tight coupling with ADK schema
* Large payloads stored in PostgreSQL
* Compliance concerns (platform storing business data)
* Limited scalability for multi-tenant usage

---

## 4. Target State Architecture

### 4.1 High-Level Overview

```plaintext
UI → Platform API → PostgreSQL (metadata)
                  → S3 (team-owned, full session data)
                  → ADK (execution engine only)
```

---

## 5. Session Storage Strategy

### 5.1 Custom Session Service

Replace ADK `DatabaseSessionService` with:

## **Custom S3SessionService (Wrapper over InMemorySessionService)**

### Responsibilities:

* Use in-memory session handling during execution
* Persist final session and event data to S3
* Avoid persistence in ADK database tables

---

### 5.2 Execution Flow

```plaintext
1. Platform creates execution (correlation_id)
2. ADK agent is invoked using InMemorySessionService
3. Events are generated in-memory during execution
4. On completion:
    - Full session + events serialized
    - Stored in team-owned S3
5. Platform updates metadata in PostgreSQL
```

---

## 6. S3 Storage Model (Team-Owned)

### 6.1 Standard Path Structure

```plaintext
s3://<team-bucket>/
  <execution_id>/
    session.json
    events.json
```

---

### 6.2 Storage Format

* Format aligns with ADK session/event structure
* Platform does not enforce transformation at storage level
* Format versioning is recommended:

```json
{
  "format": "adk_v1",
  "session": {...},
  "events": [...]
}
```

---

### 6.3 Ownership Model

| Data Type       | Owner             |
| --------------- | ----------------- |
| Session payload | Agent-owning team |
| Event payload   | Agent-owning team |
| Metadata        | Platform          |

---

## 7. Platform Metadata (PostgreSQL)

### 7.1 correlation_sessions Table

This remains the primary platform table.

---

### 7.2 Updated Schema

```sql
correlation_id (PK)
session_id

agent_name
agent_version

status
status_reason

start_time
end_time

s3_bucket
s3_prefix

additional_metadata (JSONB)
created_at
```

---

### 7.3 Purpose

* Drive UI list view
* Provide execution status
* Store pointer to S3 location
* Enable filtering and analytics

---

## 8. UI Interaction Model

### 8.1 List View

Data Source: PostgreSQL

Displays:

* correlation_id (execution_id)
* status
* status_reason
* timestamps
* agent metadata

---

### 8.2 Detail View

Data Source: S3 (via Platform API)

Flow:

```plaintext
UI → Platform API → S3 → Response to UI
```

---

### 8.3 Key Constraint

* UI must **not directly access S3**
* All access must go through platform APIs

---

## 9. Platform API Responsibilities

* Resolve S3 location using metadata
* Fetch session/event data from S3
* Perform optional transformation if required
* Enforce access control and security

---

## 10. Security and Access Model

* Team-owned S3 buckets
* Platform uses:

  * Cross-account IAM roles OR
  * Scoped access credentials
* Access validated during onboarding

---

## 11. Onboarding Requirements for Teams

Each team must provide:

* S3 bucket details
* Access role for platform
* Agreement to standard path structure
* Confirmation of data format version

---

## 12. ADK Dependency Strategy

* ADK is used only for:

  * agent execution
  * event generation
* ADK persistence (DatabaseSessionService) is not used
* Platform does not depend on ADK database schema

---

## 13. Benefits

* Compliance-aligned (no business data in platform)
* Scalable across multiple teams and agents
* Reduced database footprint
* Clear separation of responsibilities
* Flexibility to evolve platform independently of ADK

---

## 14. Risks and Mitigations

| Risk                         | Mitigation                |
| ---------------------------- | ------------------------- |
| Inconsistent S3 formats      | Enforce standard contract |
| S3 access issues             | Onboarding validation     |
| Large payload latency        | Fetch on-demand only      |
| Tight coupling to ADK format | Versioned storage format  |

---

## 15. Conclusion

The proposed architecture establishes a clear separation:

| Layer           | Responsibility              |
| --------------- | --------------------------- |
| ADK             | Execution engine            |
| PostgreSQL      | Metadata and orchestration  |
| S3 (team-owned) | Full session and event data |

This approach ensures compliance, scalability, and platform independence while enabling teams to retain ownership of their data.
