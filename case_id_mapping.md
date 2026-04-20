Build a custom `S3SessionService` for Google ADK to replace `DatabaseSessionService`.

### Context

* We provide an agent definition to an external runtime (we do NOT control runtime).
* Runtime uses Google ADK and allows us to plug in a custom `SessionService`.
* We must NOT persist any session/event data in PostgreSQL.
* Full session + events must be stored in **team-owned S3**.
* PostgreSQL is only used outside this for metadata (not part of this task).

---

### Requirements

1. **Extend ADK SessionService**

   * Implement a class `S3SessionService` compatible with ADK’s `SessionService` interface.
   * Do NOT modify ADK internals.

2. **Use InMemorySessionService internally**

   * Wrap/combine `InMemorySessionService` for all runtime operations.
   * All session + event writes go to in-memory during execution.

3. **Persist to S3 on session completion**

   * Override the appropriate lifecycle method (prefer `close_session`).
   * On session completion:

     * Fetch full session + events from `InMemorySessionService`
     * Serialize to JSON
     * Write to S3

4. **S3 Storage Format**

   * Store in:
     `<prefix>/<execution_id>/<session_id>.json`
   * Payload format:

     ```
     {
       "format": "adk_v1",
       "session": <session_object>,
       "events": <events_list>
     }
     ```

5. **Execution ID**

   * Read `execution_id` from session metadata if available
   * Fallback to session_id if not present

6. **Per-request lifecycle**

   * Each request creates a new `S3SessionService`
   * No global/shared state

7. **Failure safety**

   * Ensure persistence happens even if execution fails (use try/finally or safe close handling)

8. **Serialization**

   * Ensure session/events are JSON serializable
   * Add helper if needed

---

### Non-Goals

* No UI/API changes
* No PostgreSQL changes
* No runtime/executor modifications
* No direct S3 access from UI

---

### Deliverables

* `S3SessionService` implementation
* Clean, minimal code (no overengineering)
* Small helper functions if required (serialization, S3 write)

---

### Notes

* Keep implementation simple and readable
* Follow composition (wrap InMemorySessionService, do not reimplement logic)
* This will be used in production, so ensure correctness over cleverness
