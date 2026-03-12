Task: Implement Phase-1 Agent Onboarding API with Test Coverage

Context

We are building an internal agent platform that allows teams to create and invoke AI agents. The backend is built with FastAPI and PostgreSQL using async SQLAlchemy.

Agents will be defined using a JSON specification but stored internally as YAML.

Phase-1 goal is to implement a very simple onboarding API without validation or lifecycle management.

We also want automated tests with pytest and test coverage.

Functional Requirements

Implement a FastAPI endpoint to onboard an agent definition.

Endpoint

POST /agents

Request Example

{
"agent_name": "customer-support-agent",
"description": "Handles customer support queries",
"version": "1.0.0",
"spec": {
"model": "gemini-1.5-pro",
"instructions": "You are a helpful support agent.",
"tools": [
"kb_search",
"ticket_create"
],
"config": {
"temperature": 0.2
}
}
}

Behavior

1. Check if agent exists using agent_name
2. If not, create a new agent record
3. Convert the spec JSON to YAML using PyYAML
4. Insert a new row in agent_versions
5. Return the created agent_id and version

No validation is required in Phase-1.

Response Example

{
"agent_id": "uuid",
"agent_name": "customer-support-agent",
"version": "1.0.0",
"status": "draft"
}

Database Schema

agents

* id (uuid primary key)
* name (unique)
* description
* created_at

agent_versions

* id (uuid primary key)
* agent_id (foreign key)
* version
* yaml_definition (text)
* status (default = draft)
* created_at

Implementation Requirements

Use async SQLAlchemy.
Use asyncpg PostgreSQL driver.
Use PyYAML to convert JSON spec → YAML.

Project Structure

backend/
server/
api/
agents_api.py
models/
agent_models.py

registry/
agent_registry.py

db/
postgres.py
models.py

tests/
conftest.py
test_agents_api.py
test_agent_registry.py

Dependencies

fastapi
sqlalchemy[asyncio]
asyncpg
pytest
pytest-asyncio
httpx
pyyaml

Implementation Steps

1. Database Models

File
db/models.py

Tables
Agent
AgentVersion

2. Agent Registry Service

File
registry/agent_registry.py

Methods

create_agent(name, description)
get_agent_by_name(name)
create_version(agent_id, version, yaml_definition)

3. Pydantic Models

File
server/models/agent_models.py

AgentCreateRequest
AgentCreateResponse

4. API Route

File
server/api/agents_api.py

POST /agents

Flow

receive request
↓
check if agent exists
↓
create agent if needed
↓
convert spec to YAML
↓
store version
↓
return response

Testing Requirements

Use pytest and pytest-asyncio.

Tests must include:

API Tests
tests/test_agents_api.py

* should create a new agent successfully
* should create a new version if agent exists
* response should contain agent_id and version
* YAML should be stored in DB

Registry Tests
tests/test_agent_registry.py

* create_agent creates a row
* get_agent_by_name retrieves agent
* create_version stores version correctly

Test Setup

Use a separate test database or SQLite in-memory DB.

Provide a test fixture for:

* async database session
* FastAPI test client using httpx AsyncClient

Example test

async def test_create_agent(client):
payload = {
"agent_name": "support-agent",
"version": "1.0.0",
"spec": {
"model": "gemini-1.5-pro",
"instructions": "You are a helpful support agent"
}
}

```
response = await client.post("/agents", json=payload)

assert response.status_code == 200
assert response.json()["agent_name"] == "support-agent"
```

Coverage

Tests should cover:

API endpoint
registry service
YAML serialization

Target coverage: at least 80%.

Notes

Do not implement validation.
Do not instantiate Google ADK agents.
Only store the YAML specification.

The goal of Phase-1 is a minimal but production-quality onboarding API with test coverage.


Additional Requirement: Database Table Creation Script

Since this is Phase-1 of the platform, we do not want to introduce migrations yet.

Instead, generate a SQL script that creates the required tables.

Create a file:

db/schema.sql


---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

Task: Implement Phase-1 Agent Invocation API with DatabaseSessionService Support

Context

We are building an internal agent platform that allows teams to onboard and invoke AI agents. The backend is built using FastAPI and PostgreSQL. Agent definitions are stored as YAML in the database.

We already implemented the onboarding API which stores agent YAML definitions in the database.

Now we want to implement a simple agent invocation endpoint.

Our platform already has a common component called LLMAgent which wraps the Google ADK agent and creates agents based on configuration. Currently LLMAgent uses InMemorySessionService. We want to extend this component to support DatabaseSessionService so that agent sessions can be persisted in the database.

Phase-1 goal is to keep the implementation simple.

Requirements

1. Implement a new API endpoint to invoke an agent.

Endpoint

POST /agents/invoke

Request Example

{
"agent_name": "customer-support-agent",
"version": "1.0.0",
"input": "My payment failed",
"session_id": "abc123",
"params": {}
}

Fields

agent_name: name of the agent
version: agent version to execute
input: user input to the agent
session_id: session identifier
params: optional runtime parameters

Response Example

{
"response": "I'm sorry to hear your payment failed. Let me help you troubleshoot."
}

High-Level Flow

API receives invoke request
↓
Load agent YAML definition from database using agent_name and version
↓
Convert YAML to Python dictionary
↓
Use LLMAgent to construct the ADK agent instance
↓
Use DatabaseSessionService instead of InMemorySessionService
↓
Execute the agent with the input
↓
Return agent response

Database Lookup

Use existing AgentRegistry to fetch agent version.

Method

get_version(agent_name, version)

The YAML definition stored in the database should be parsed using PyYAML.

LLMAgent Update

The existing LLMAgent component currently uses InMemorySessionService.

Extend LLMAgent so that it can optionally accept a session service.

Current behavior

LLMAgent(config)

New behavior

LLMAgent(config, session_service=None)

If session_service is provided, use it when creating the ADK agent.

If not provided, fallback to InMemorySessionService.

Example

session_service = DatabaseSessionService(...)

agent = LLMAgent(
config=agent_config,
session_service=session_service
)

Session Service

Use DatabaseSessionService provided by Google ADK.

The session service should be initialized once and reused.

Example

session_service = DatabaseSessionService(db_connection)

Project Structure

backend/
server/
api/
agents_api.py
invoke_api.py
models/
agent_models.py

registry/
agent_registry.py

runtime/
agent_executor.py

db/
postgres.py

tests/
test_invoke_api.py

Implementation Steps

1. Create request/response models

server/models/agent_models.py

InvokeAgentRequest
InvokeAgentResponse

2. Implement invoke endpoint

server/api/invoke_api.py

POST /agents/invoke

Responsibilities

* fetch agent spec from DB
* parse YAML
* construct LLMAgent
* run agent

3. Extend LLMAgent

Add support for optional session_service parameter.

If provided, pass it to the underlying ADK agent.

If not provided, default to InMemorySessionService.

4. Implement agent execution helper

runtime/agent_executor.py

Responsibilities

* construct LLMAgent
* run agent
* return result

Testing Requirements

Create API tests using pytest.

tests/test_invoke_api.py

Test cases

* invoke agent successfully
* response contains agent output
* correct agent version loaded
* session_id passed correctly

Mock LLMAgent execution to avoid calling real LLMs.

Example

mock_llm_agent.run → return fixed response.

Dependencies

fastapi
sqlalchemy[asyncio]
asyncpg
pyyaml
pytest
pytest-asyncio
httpx
google-adk

Notes

Keep the implementation simple.

Do not implement caching in this phase.

Do not implement agent lifecycle management yet.

Focus only on:

* loading agent definition
* constructing LLMAgent
* executing the agent
* returning the response.

If session exists → continue conversation
If not → new conversation

Request Example (New Session)
{
  "agent_name": "customer-support-agent",
  "version": "1.0.0",
  "caller_system": "payments-service",
  "input": "My payment failed"
}
Server Logic

Flow:

request received
     │
check session_id
     │
if missing
     │
generate new session_id
     │
load agent spec
     │
create LLMAgent
     │
execute agent
     │
return response + session_id
Response
{
  "session_id": "generated-session-id",
  "response": "I'm sorry your payment failed. Let me help you troubleshoot."
}

Now the client can reuse the session:

{
  "session_id": "generated-session-id",
  "input": "It says insufficient funds"
}
Pydantic Request Model
from pydantic import BaseModel
from typing import Optional, Dict, Any


class InvokeAgentRequest(BaseModel):
    agent_name: str
    version: str
    caller_system: str
    input: str
    session_id: Optional[str] = None
    params: Optional[Dict[str, Any]] = None
    metadata: Optional[Dict[str, Any]] = None
Response Model
class InvokeAgentResponse(BaseModel):
    session_id: str
    response: str
