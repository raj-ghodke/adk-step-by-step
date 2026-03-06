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
