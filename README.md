# Welcome to GitHub Desktop!

This is your README. READMEs are where you can communicate what your project is and how to use it.

Write your name on line 6, save it, and then head back to GitHub Desktop.

Title

Introduce framework registry and base classes for agents and tools without modifying execution flow or breaking existing functionality

Context

We have a production system with:

Google ADK agents working

MCP tools working

HITL decorator working

CopilotKit UI working

FastAPI backend working

All APIs in use and must remain unchanged

We want to introduce a new framework layer to support future plug-and-play agent and tool development.

This phase must NOT change execution behavior.

Framework will only:

Register agents

Register tools

Provide base classes

Provide loaders

It will NOT execute agents/tools yet.

Critical Requirement

Do NOT modify:

Existing endpoints

Existing agent execution flow

Existing tool execution flow

HITL decorator behavior

CopilotKit integration

Framework must run passively alongside existing system.

No behavior change allowed.

Task 1: Create Framework Folder Structure

Create new folder at backend root:

framework/

Structure:

framework/
├── __init__.py
│
├── agent/
│   ├── __init__.py
│   ├── base_agent.py
│   └── agent_registry.py
│
├── tool/
│   ├── __init__.py
│   └── tool_registry.py
│
├── runtime/
│   └── context.py
│
└── loader/
    ├── agent_loader.py
    └── tool_loader.py

Do NOT modify existing folders.

Task 2: Implement Tool Registry

Create file:

framework/tool/tool_registry.py

Implementation:

class ToolRegistry:

    def __init__(self):

        self._tools = {}


    def register(self, name: str, tool):

        if name in self._tools:
            return

        print(f"[Framework] Registering tool: {name}")

        self._tools[name] = tool


    def get(self, name: str):

        return self._tools.get(name)


    def list(self):

        return list(self._tools.keys())


tool_registry = ToolRegistry()

This must NOT execute tools.

Only register.

Task 3: Implement Agent Registry

Create:

framework/agent/agent_registry.py

Implementation:

class AgentRegistry:

    def __init__(self):

        self._agents = {}


    def register(self, name: str, agent):

        if name in self._agents:
            return

        print(f"[Framework] Registering agent: {name}")

        self._agents[name] = agent


    def get(self, name: str):

        return self._agents.get(name)


    def list(self):

        return list(self._agents.keys())


agent_registry = AgentRegistry()

Passive registry only.

Task 4: Implement BaseAgent Class

Create:

framework/agent/base_agent.py

Implementation:

class BaseAgent:

    name = None

    tools = []


    def __init__(self):

        if self.name:

            from framework.agent.agent_registry import agent_registry

            agent_registry.register(self.name, self)


    async def run(self, input, context):

        raise NotImplementedError

This does NOT modify existing agents.

Existing agents do not need to inherit yet.

Task 5: Implement Context Object

Create:

framework/runtime/context.py

Implementation:

class AgentContext:

    def __init__(
        self,
        session_id: str,
        agent_name: str = None,
        metadata: dict = None
    ):

        self.session_id = session_id
        self.agent_name = agent_name
        self.metadata = metadata or {}

Do NOT integrate yet.

Task 6: Implement Tool Loader

Create:

framework/loader/tool_loader.py

Implementation:

import os
import importlib


def load_tools():

    tools_dir = "tools"

    if not os.path.exists(tools_dir):
        return

    for file in os.listdir(tools_dir):

        if file.endswith(".py") and not file.startswith("__"):

            module = f"tools.{file[:-3]}"

            print(f"[Framework] Loading tool module: {module}")

            importlib.import_module(module)

This ensures decorators execute and register tools.

Task 7: Implement Agent Loader

Create:

framework/loader/agent_loader.py

Implementation:

import os
import importlib


def load_agents():

    agents_dir = "agents"

    if not os.path.exists(agents_dir):
        return

    for folder in os.listdir(agents_dir):

        path = os.path.join(agents_dir, folder)

        if os.path.isdir(path):

            module = f"agents.{folder}"

            print(f"[Framework] Loading agent module: {module}")

            importlib.import_module(module)
Task 8: Modify Existing Tool Decorator (Safe Passive Registration)

Locate existing tool decorator.

Add tool registration WITHOUT changing behavior.

Example:

from framework.tool.tool_registry import tool_registry

def existing_tool_decorator(...):

    def decorator(func):

        tool_registry.register(func.__name__, func)

        async def wrapper(*args, **kwargs):

            return await func(*args, **kwargs)

        return wrapper

    return decorator

IMPORTANT:

Do NOT change execution logic.

Only register.

Task 9: Register Existing Agents (Passive)

Locate existing agent initialization.

After agent creation, register agent.

Example:

from framework.agent.agent_registry import agent_registry

agent_registry.register("recon_agent", recon_agent_instance)

This must NOT change execution.

Task 10: Initialize Framework on Startup

Locate backend startup code.

Add:

from framework.loader.tool_loader import load_tools
from framework.loader.agent_loader import load_agents

load_tools()
load_agents()

Must NOT affect runtime behavior.

Task 11: Add Debug Logging

Add logs:

print("[Framework] Tool registered:", name)
print("[Framework] Agent registered:", name)

Helps validation.

Task 12: Validation Requirement

After implementation, verify:

Existing functionality works exactly same:

Agents execute correctly

Tools execute correctly

HITL approval works

Resume works

CopilotKit works

MCP tools work

No behavior changes allowed.

Expected Result

Framework exists and logs:

[Framework] Loading tool module: tools.resolution_note_tool
[Framework] Registering tool: resolution_note_tool
[Framework] Loading agent module: agents.recon_agent
[Framework] Registering agent: recon_agent

But execution still uses existing system.

Framework is passive.

Deliverables

Create:

framework/

With all files listed above.

Modify existing decorators and agent initialization ONLY to register.

Do NOT modify execution logic.

DO NOT PROCEED TO RUNTIME MIGRATION YET

Runtime migration will happen in later phase.

This phase only builds foundation safely.



-------------------------------------------------------------------------------------------------------------------------------------------------------------------



Title

Introduce AgentRuntime wrapper and route execution through framework while preserving exact existing behavior and API contracts

Context

Phase 1–3 are already completed.

Framework components already exist:

framework/
├── agent_registry.py
├── tool_registry.py
├── base_agent.py
├── tool_loader.py
├── agent_loader.py
└── context.py

Currently:

Existing agents execute directly

Framework registers agents and tools passively

Framework does NOT control execution yet

Everything works correctly.

Now we want framework runtime to wrap execution safely.

Critical Requirements

DO NOT break:

Existing API endpoints

Existing request/response formats

CopilotKit integration

HITL approval flow

MCP tools execution

Session handling

Execution behavior must remain identical.

Framework runtime must initially act as a passthrough wrapper.

Must support instant rollback.

Task 1: Create AgentRuntime Wrapper

Create file:

framework/runtime/agent_runtime.py

Implementation:

from framework.agent.agent_registry import agent_registry
from framework.runtime.context import AgentContext


class AgentRuntime:

    def __init__(self):

        print("[Framework] AgentRuntime initialized")


    async def run(

        self,
        agent_name: str,
        input_data,
        session_id: str,
        metadata: dict = None

    ):

        print(f"[Framework] Running agent: {agent_name}")

        agent = agent_registry.get(agent_name)

        if not agent:

            raise Exception(
                f"Agent not found in registry: {agent_name}"
            )

        context = AgentContext(
            session_id=session_id,
            agent_name=agent_name,
            metadata=metadata or {}
        )

        # IMPORTANT: call existing agent run method
        result = await agent.run(input_data, context)

        return result


agent_runtime = AgentRuntime()

This wrapper must not change execution logic.

Only wraps it.

Task 2: Add Runtime Feature Flag (Critical Safety Requirement)

Create file:

framework/config/runtime_config.py

Implementation:

USE_FRAMEWORK_RUNTIME = True

This allows rollback instantly.

Task 3: Modify Existing Agent Execution Endpoint Safely

Locate existing endpoint:

Example:

@app.post("/agent/run")
async def run_agent(request: Request):

Modify safely:

Before:

result = await agent.run(
    request.input,
    request.session_id
)

After:

from framework.config.runtime_config import USE_FRAMEWORK_RUNTIME

if USE_FRAMEWORK_RUNTIME:

    from framework.runtime.agent_runtime import agent_runtime

    result = await agent_runtime.run(
        agent_name=request.agent_name,
        input_data=request.input,
        session_id=request.session_id,
        metadata=request.metadata if hasattr(request, "metadata") else None
    )

else:

    # fallback to existing execution
    result = await agent.run(
        request.input,
        request.session_id
    )

DO NOT modify request or response format.

Task 4: Ensure Existing Agents Still Work

Existing agents may use either:

async def run(self, input, session_id)

or:

async def run(self, input, context)

Add compatibility wrapper in runtime:

Update runtime:

import inspect

if len(inspect.signature(agent.run).parameters) == 2:

    result = await agent.run(input_data, context)

else:

    result = await agent.run(input_data, session_id)

This ensures backward compatibility.

Task 5: Add Execution Logging

Add logs:

print(f"[Framework] Executing agent via runtime: {agent_name}")
print(f"[Framework] Session ID: {session_id}")

Helps debugging.

Task 6: Validate Tool Execution Still Works

Tools must still execute via:

existing decorator

MCP tools

HITL approval flow

Runtime must NOT intercept tool execution yet.

Only agent execution wrapped.

Task 7: Validate HITL Still Works

Test flow:

User message
Agent executes
Tool requires approval
CopilotKit shows approval
User approves
Resume endpoint works
Agent resumes

All must work exactly same.

Task 8: Validate CopilotKit Integration

CopilotKit expects response format:

{
  "content": "...",
  "actions": [...]
}

Runtime must NOT modify this.

Return result exactly as agent returns.

Task 9: Add Startup Initialization

Ensure loaders run at startup:

from framework.loader.tool_loader import load_tools
from framework.loader.agent_loader import load_agents

load_tools()
load_agents()
Task 10: Validation Checklist (Mandatory)

Verify:

Agent execution works:

POST /agent/run

Tool execution works.

MCP tools work.

HITL approval works.

Resume works.

CopilotKit UI works.

No errors in logs.

Expected Logs After Implementation
[Framework] AgentRuntime initialized
[Framework] Running agent: recon_agent
[Framework] Executing agent via runtime: recon_agent
Rollback Plan (Must Work Instantly)

If issues occur, change:

USE_FRAMEWORK_RUNTIME = False

No other changes needed.

System must immediately use old execution path.

Deliverables

Create:

framework/runtime/agent_runtime.py
framework/config/runtime_config.py

Modify execution endpoint safely using feature flag.

Ensure backward compatibility.

DO NOT MODIFY:

Tools

HITL decorator

CopilotKit integration

MCP integration

This phase only wraps agent execution.

After Completion

Framework runtime becomes execution entry point safely.

No behavior changes.



--------------------------------------------------------------------------------------------------------------------------------------------------------------------


Title

Refactor tool execution to use framework ToolRegistry while preserving existing tool behavior, MCP integration, HITL decorator, and CopilotKit flow

Context

Phase 1–4 already completed.

Framework now has:

ToolRegistry (passive registration)

AgentRegistry

AgentRuntime wrapping agent execution

Loaders auto-registering tools and agents

Feature flag controlling runtime usage

Current state:

Agents execute via framework runtime

Tools still execute directly via existing code

HITL decorator already working

MCP tools already integrated

CopilotKit integration working

Goal now:

Enable framework to execute tools via ToolRegistry while preserving all existing execution paths.

Critical Requirements

DO NOT break:

Existing tools

Existing MCP tools

HITL decorator functionality

CopilotKit approval flow

Existing agents

Existing API contract

All existing tools must continue working unchanged.

Framework must support both:

Framework-registered tools

Legacy tools

Task 1: Add Framework Tool Execution Method

Update:

framework/tool/tool_registry.py

Add execution method:

async def execute(
    self,
    tool_name: str,
    input_data,
    context
):

    tool = self.get(tool_name)

    if not tool:

        raise Exception(
            f"Tool not found in registry: {tool_name}"
        )

    print(f"[Framework] Executing tool via registry: {tool_name}")

    return await tool(input_data, context)

Full updated class:

class ToolRegistry:

    def __init__(self):

        self._tools = {}

    def register(self, name, tool):

        if name in self._tools:
            return

        print(f"[Framework] Registering tool: {name}")

        self._tools[name] = tool

    def get(self, name):

        return self._tools.get(name)

    async def execute(self, name, input_data, context):

        tool = self.get(name)

        if not tool:
            raise Exception(f"Tool not found: {name}")

        return await tool(input_data, context)

tool_registry = ToolRegistry()
Task 2: Add Tool Execution Method in BaseAgent

Update:

framework/agent/base_agent.py

Add:

from framework.tool.tool_registry import tool_registry


class BaseAgent:

    name = None

    tools = []


    async def execute_tool(
        self,
        tool_name: str,
        input_data,
        context
    ):

        print(f"[Framework] Agent calling tool: {tool_name}")

        return await tool_registry.execute(
            tool_name,
            input_data,
            context
        )

This enables plug-and-play tool execution.

Task 3: Add Tool Execution Feature Flag (Critical)

Update:

framework/config/runtime_config.py

Add:

USE_FRAMEWORK_TOOL_EXECUTION = False

Default must be False initially.

Allows safe rollout.

Task 4: Update Existing Tool Call Sites Safely

Locate existing tool execution code.

Example:

Before:

result = await resolution_tool(input_data, context)

After:

from framework.config.runtime_config import (
    USE_FRAMEWORK_TOOL_EXECUTION
)

if USE_FRAMEWORK_TOOL_EXECUTION:

    result = await self.execute_tool(
        "resolution_tool",
        input_data,
        context
    )

else:

    result = await resolution_tool(
        input_data,
        context
    )

This preserves backward compatibility.

Task 5: Ensure HITL Decorator Continues Working

Existing decorator already wraps tool.

No change required.

Framework execution must call wrapped tool function.

DO NOT modify decorator behavior.

Decorator already registered tool via ToolRegistry in Phase 1–3.

Task 6: Ensure MCP Tools Continue Working

If MCP tools executed via wrapper like:

await mcp_client.call_tool(...)

Register MCP tools as well:

Example:

tool_registry.register(
    "mcp_resolution_tool",
    mcp_tool_wrapper
)

DO NOT modify MCP client logic.

Framework only provides optional execution path.

Task 7: Add Tool Execution Logging

Add logs:

print(f"[Framework] Executing tool: {tool_name}")
print(f"[Framework] Tool input: {input_data}")

Helps debugging.

Task 8: Validate Tool Execution Flow

Test:

Normal tool execution:

User message
Agent executes
Tool executes
Response returned

Test HITL tool:

Tool requires approval
Approval shown in CopilotKit
Approve works
Resume works

Test MCP tool:

Agent calls MCP tool
MCP tool executes
Response returned

All must work unchanged.

Task 9: Enable Framework Tool Execution for ONE Tool Only (Safe Rollout)

Example:

USE_FRAMEWORK_TOOL_EXECUTION = True

Test resolution_note_tool only.

Do NOT migrate all tools at once.

Task 10: Verify CopilotKit Integration

Ensure CopilotKit still receives:

{
  "content": "...",
  "actions": [...]
}

Framework must not modify response format.

Expected Logs After Migration
[Framework] Executing tool via registry: resolution_note_tool
[Framework] Agent calling tool: resolution_note_tool
Rollback Plan

If issues occur:

USE_FRAMEWORK_TOOL_EXECUTION = False

System immediately returns to legacy execution.

No code rollback needed.

Deliverables

Update:

framework/tool/tool_registry.py
framework/agent/base_agent.py
framework/config/runtime_config.py

Modify existing tool call sites safely.

Add feature flag.

Ensure backward compatibility.

Acceptance Criteria

Must confirm:

Agents work

Tools work

HITL works

MCP tools work

CopilotKit works

Resume works

No API changes

No response format changes

After Phase 5 Completion

Framework fully supports:

Plug-and-play tools:

@tool(name="new_tool")
async def new_tool(...)

Agent can use immediately without wiring.


------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

itle

Add configuration-driven HITL, tool approval policies, and tool behavior via YAML config without breaking existing decorator-based HITL or tool execution

Context

Phase 1–5 completed.

Framework now supports:

AgentRegistry

ToolRegistry

AgentRuntime

Tool execution via registry (feature-flag controlled)

HITL decorator working

MCP tools working

CopilotKit integration working

Currently HITL is controlled by decorator only.

Example:

@requires_approval(
    message="Approve resolution note"
)
async def resolution_tool(...):

Goal now:

Allow HITL and tool behavior to be controlled via config instead of code.

This allows:

Enabling/disabling approval without code change

Changing approval messages without code change

Adding policies centrally

Enterprise governance

Decorator must continue working.

Config overrides decorator when present.

Task 1: Create Tool Configuration File

Create:

framework/config/tools.yaml

Example content:

tools:

  resolution_note_tool:

    requires_approval: true

    approval_message: "Please approve resolution note"

    enabled: true

  reconciliation_tool:

    requires_approval: false

    enabled: true

  mcp_resolution_tool:

    requires_approval: true

    approval_message: "Approve MCP resolution"
Task 2: Create ToolConfig Loader

Create file:

framework/config/tool_config_loader.py

Implementation:

import yaml
import os


class ToolConfigLoader:

    def __init__(self):

        self.config = {}

        self.load()


    def load(self):

        path = "framework/config/tools.yaml"

        if not os.path.exists(path):

            print("[Framework] No tools.yaml found")

            return

        with open(path, "r") as f:

            data = yaml.safe_load(f)

            self.config = data.get("tools", {})

        print("[Framework] Tool config loaded")


    def get(self, tool_name):

        return self.config.get(tool_name, {})


tool_config_loader = ToolConfigLoader()
Task 3: Integrate Config into ToolRegistry Execution

Update:

framework/tool/tool_registry.py

Modify execute method:

from framework.config.tool_config_loader import tool_config_loader
from framework.hitl.hitl_manager import hitl_manager


async def execute(
    self,
    tool_name,
    input_data,
    context
):

    tool = self.get(tool_name)

    if not tool:

        raise Exception(f"Tool not found: {tool_name}")


    # load config
    config = tool_config_loader.get(tool_name)

    requires_approval = config.get(
        "requires_approval",
        False
    )

    approval_message = config.get(
        "approval_message",
        f"Approve tool execution: {tool_name}"
    )


    if requires_approval:

        print(f"[Framework] HITL required for tool: {tool_name}")

        approval_id = hitl_manager.create_approval(
            session_id=context.session_id,
            agent_name=context.agent_name,
            tool_name=tool_name,
            payload=input_data
        )

        return {

            "content": approval_message,

            "actions": [

                {

                    "name": "requestApproval",

                    "arguments": {

                        "approvalId": approval_id,
                        "message": approval_message

                    }

                }

            ]

        }


    # normal execution
    return await tool(input_data, context)
Task 4: Ensure Backward Compatibility with Decorator

Decorator already handles approval.

We must support both decorator and config.

Decorator logic should remain unchanged.

Framework config logic must only apply if tool is executed via registry.

Decorator approval takes precedence.

Do NOT remove decorator approval logic.

Task 5: Add Tool Enable/Disable Support

Extend execute method:

enabled = config.get("enabled", True)

if not enabled:

    raise Exception(
        f"Tool disabled via config: {tool_name}"
    )
Task 6: Add Config Reload Capability

Add reload method:

def reload(self):

    self.load()

Allows hot reload later.

Task 7: Add Logging

Add logs:

print(f"[Framework] Tool config: {tool_name} -> {config}")
print(f"[Framework] Approval required via config: {tool_name}")
Task 8: Validate Approval Flow via Config

Test scenario:

Remove decorator from resolution tool.

Add config:

resolution_note_tool:

  requires_approval: true

Verify:

CopilotKit still shows approval.

Resume still works.

Task 9: Validate MCP Tools

Add MCP tool config:

mcp_resolution_tool:

  requires_approval: true

Verify approval works.

Task 10: Validate Existing Decorator Still Works

Existing decorator tools must still require approval.

Framework must NOT break decorator approval.

Task 11: Feature Flag for Config-Driven HITL (Safety)

Update:

framework/config/runtime_config.py

Add:

USE_CONFIG_DRIVEN_HITL = True

Use in registry:

if USE_CONFIG_DRIVEN_HITL and requires_approval:
Expected Logs
[Framework] Tool config loaded
[Framework] Tool config: resolution_note_tool -> {...}
[Framework] HITL required for tool: resolution_note_tool
Rollback Plan

If issues occur:

USE_CONFIG_DRIVEN_HITL = False

System uses decorator only.

Deliverables

Create:

framework/config/tools.yaml
framework/config/tool_config_loader.py

Modify:

framework/tool/tool_registry.py
framework/config/runtime_config.py
Acceptance Criteria

Must confirm:

Agents work

Tools work

HITL works via config

HITL works via decorator

CopilotKit works

Resume works

MCP tools work

No API changes

Result After Phase 6

Your system now supports:

Fully config-driven tool behavior:

tools:

  resolution_note_tool:

    requires_approval: true

No code changes required.


----------------------------------------------------------------------------------------------------------------------------------------------------------------------------

itle

Add configuration-driven HITL, tool approval policies, and tool behavior via YAML config without breaking existing decorator-based HITL or tool execution

Context

Phase 1–5 completed.

Framework now supports:

AgentRegistry

ToolRegistry

AgentRuntime

Tool execution via registry (feature-flag controlled)

HITL decorator working

MCP tools working

CopilotKit integration working

Currently HITL is controlled by decorator only.

Example:

@requires_approval(
    message="Approve resolution note"
)
async def resolution_tool(...):

Goal now:

Allow HITL and tool behavior to be controlled via config instead of code.

This allows:

Enabling/disabling approval without code change

Changing approval messages without code change

Adding policies centrally

Enterprise governance

Decorator must continue working.

Config overrides decorator when present.

Task 1: Create Tool Configuration File

Create:

framework/config/tools.yaml

Example content:

tools:

  resolution_note_tool:

    requires_approval: true

    approval_message: "Please approve resolution note"

    enabled: true

  reconciliation_tool:

    requires_approval: false

    enabled: true

  mcp_resolution_tool:

    requires_approval: true

    approval_message: "Approve MCP resolution"
Task 2: Create ToolConfig Loader

Create file:

framework/config/tool_config_loader.py

Implementation:

import yaml
import os


class ToolConfigLoader:

    def __init__(self):

        self.config = {}

        self.load()


    def load(self):

        path = "framework/config/tools.yaml"

        if not os.path.exists(path):

            print("[Framework] No tools.yaml found")

            return

        with open(path, "r") as f:

            data = yaml.safe_load(f)

            self.config = data.get("tools", {})

        print("[Framework] Tool config loaded")


    def get(self, tool_name):

        return self.config.get(tool_name, {})


tool_config_loader = ToolConfigLoader()
Task 3: Integrate Config into ToolRegistry Execution

Update:

framework/tool/tool_registry.py

Modify execute method:

from framework.config.tool_config_loader import tool_config_loader
from framework.hitl.hitl_manager import hitl_manager


async def execute(
    self,
    tool_name,
    input_data,
    context
):

    tool = self.get(tool_name)

    if not tool:

        raise Exception(f"Tool not found: {tool_name}")


    # load config
    config = tool_config_loader.get(tool_name)

    requires_approval = config.get(
        "requires_approval",
        False
    )

    approval_message = config.get(
        "approval_message",
        f"Approve tool execution: {tool_name}"
    )


    if requires_approval:

        print(f"[Framework] HITL required for tool: {tool_name}")

        approval_id = hitl_manager.create_approval(
            session_id=context.session_id,
            agent_name=context.agent_name,
            tool_name=tool_name,
            payload=input_data
        )

        return {

            "content": approval_message,

            "actions": [

                {

                    "name": "requestApproval",

                    "arguments": {

                        "approvalId": approval_id,
                        "message": approval_message

                    }

                }

            ]

        }


    # normal execution
    return await tool(input_data, context)
Task 4: Ensure Backward Compatibility with Decorator

Decorator already handles approval.

We must support both decorator and config.

Decorator logic should remain unchanged.

Framework config logic must only apply if tool is executed via registry.

Decorator approval takes precedence.

Do NOT remove decorator approval logic.

Task 5: Add Tool Enable/Disable Support

Extend execute method:

enabled = config.get("enabled", True)

if not enabled:

    raise Exception(
        f"Tool disabled via config: {tool_name}"
    )
Task 6: Add Config Reload Capability

Add reload method:

def reload(self):

    self.load()

Allows hot reload later.

Task 7: Add Logging

Add logs:

print(f"[Framework] Tool config: {tool_name} -> {config}")
print(f"[Framework] Approval required via config: {tool_name}")
Task 8: Validate Approval Flow via Config

Test scenario:

Remove decorator from resolution tool.

Add config:

resolution_note_tool:

  requires_approval: true

Verify:

CopilotKit still shows approval.

Resume still works.

Task 9: Validate MCP Tools

Add MCP tool config:

mcp_resolution_tool:

  requires_approval: true

Verify approval works.

Task 10: Validate Existing Decorator Still Works

Existing decorator tools must still require approval.

Framework must NOT break decorator approval.

Task 11: Feature Flag for Config-Driven HITL (Safety)

Update:

framework/config/runtime_config.py

Add:

USE_CONFIG_DRIVEN_HITL = True

Use in registry:

if USE_CONFIG_DRIVEN_HITL and requires_approval:
Expected Logs
[Framework] Tool config loaded
[Framework] Tool config: resolution_note_tool -> {...}
[Framework] HITL required for tool: resolution_note_tool
Rollback Plan

If issues occur:

USE_CONFIG_DRIVEN_HITL = False

System uses decorator only.

Deliverables

Create:

framework/config/tools.yaml
framework/config/tool_config_loader.py

Modify:

framework/tool/tool_registry.py
framework/config/runtime_config.py
Acceptance Criteria

Must confirm:

Agents work

Tools work

HITL works via config

HITL works via decorator

CopilotKit works

Resume works

MCP tools work

No API changes

Result After Phase 6

Your system now supports:

Fully config-driven tool behavior:

tools:

  resolution_note_tool:

    requires_approval: true

No code changes required.

---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------


Title

Extend existing HITL system to support role-based, multi-level approvals with escalation and audit trail, without breaking existing approval flow or CopilotKit integration

Context

Framework currently supports:

ToolRegistry execution

Config-driven HITL via tools.yaml

Decorator-based HITL

CopilotKit approval UI working

Approval stored in SQLite

Resume endpoint working

Currently approval model is:

1 tool → 1 approval → approved/rejected

We need enterprise model:

tool → approval workflow → multiple levels → assigned roles → final approval

Example:

Level 1: Ops approval
Level 2: Manager approval
Level 3: Auto execute

This must work without breaking existing single-approval tools.

Task 1: Update tools.yaml to Support Multi-Level Approval

Modify:

framework/config/tools.yaml

New structure:

tools:

  resolution_note_tool:

    enabled: true

    approval_workflow:

      - level: 1
        role: "ops"
        required: true

      - level: 2
        role: "manager"
        required: true

  reconciliation_tool:

    enabled: true

    approval_workflow:

      - level: 1
        role: "ops"
        required: true

  low_risk_tool:

    enabled: true

    auto_approve: true

Backward compatibility:

Still support:

requires_approval: true
Task 2: Extend Approval Database Schema

Modify approvals table:

Add columns:

ALTER TABLE approvals ADD COLUMN level INTEGER DEFAULT 1;
ALTER TABLE approvals ADD COLUMN role TEXT;
ALTER TABLE approvals ADD COLUMN status TEXT;
ALTER TABLE approvals ADD COLUMN created_by TEXT;
ALTER TABLE approvals ADD COLUMN updated_at TIMESTAMP;

Create new table:

CREATE TABLE approval_workflows (

    id TEXT PRIMARY KEY,

    approval_id TEXT,

    level INTEGER,

    role TEXT,

    status TEXT,

    assigned_to TEXT,

    created_at TIMESTAMP,

    updated_at TIMESTAMP

);
Task 3: Extend HITL Manager

Modify:

framework/hitl/hitl_manager.py

Add method:

def create_workflow(
    self,
    approval_id,
    workflow_config
):

Implementation:

import uuid
import sqlite3


def create_workflow(self, approval_id, workflow_config):

    conn = sqlite3.connect(self.db)

    for step in workflow_config:

        workflow_id = str(uuid.uuid4())

        conn.execute("""

        INSERT INTO approval_workflows

        (id, approval_id, level, role, status)

        VALUES (?, ?, ?, ?, ?)

        """, (

            workflow_id,
            approval_id,
            step["level"],
            step["role"],
            "PENDING"

        ))

    conn.commit()
    conn.close()
Task 4: Modify ToolRegistry to Initiate Workflow

Update:

framework/tool/tool_registry.py

Replace approval logic:

Before:

approval_id = hitl_manager.create_approval(...)

After:

approval_id = hitl_manager.create_approval(...)

workflow_config = config.get("approval_workflow")

if workflow_config:

    hitl_manager.create_workflow(
        approval_id,
        workflow_config
    )
Task 5: Update Approval Endpoint to Support Multi-Level

Modify:

POST /approvals/{approval_id}/approve

New logic:

def approve(approval_id, user_role):

    current_step = get_current_pending_step(approval_id)

    if current_step.role != user_role:

        raise Exception("Unauthorized approval")

    mark_step_approved(current_step)

    next_step = get_next_step(approval_id)

    if next_step:

        assign_to_role(next_step.role)

        return {
            "status": "NEXT_LEVEL",
            "role": next_step.role
        }

    else:

        mark_approval_complete(approval_id)

        return {
            "status": "APPROVED"
        }
Task 6: Update Resume Logic

Modify resume endpoint:

POST /agents/resume/{approval_id}

New logic:

if not hitl_manager.is_fully_approved(approval_id):

    return {
        "content": "Approval still pending"
    }

# execute tool now
Task 7: Update CopilotKit Action to Include Role

Modify CopilotKit action payload:

{
  "approvalId": "...",
  "message": "...",
  "requiredRole": "manager",
  "level": 2
}

CopilotKit UI must display:

Approval required: Manager
Level: 2
Task 8: Add User Role Support

Create file:

framework/auth/user_context.py
class UserContext:

    def __init__(self, user_id, role):

        self.user_id = user_id

        self.role = role

Approval endpoint must validate role.

Task 9: Maintain Backward Compatibility

If tools.yaml contains:

requires_approval: true

Framework must behave as single-level approval.

Do NOT break existing approval tools.

Task 10: Add Approval Audit Trail

Extend approval_workflows table to track:

approved_by TEXT,
approved_at TIMESTAMP

Record every approval action.

Task 11: Add Logging

Add logs:

print(f"[Framework] Approval created: {approval_id}")
print(f"[Framework] Approval level completed: {level}")
print(f"[Framework] Approval fully completed")
Task 12: Feature Flag (Critical)

Update:

framework/config/runtime_config.py

Add:

USE_MULTI_LEVEL_APPROVAL = True

If False → fallback to old behavior.

Task 13: Validation Scenarios

Test:

Single-level approval tool → works

Multi-level tool:

Ops approves → Manager approves → Tool executes

CopilotKit UI must show correct approval role.

Resume must only work after final approval.

Deliverables

Modify:

framework/config/tools.yaml
framework/hitl/hitl_manager.py
framework/tool/tool_registry.py
framework/api/approval_routes.py
framework/api/resume_routes.py

Create:

framework/auth/user_context.py

Update database schema safely.

Acceptance Criteria

Must confirm:

Single-level approval still works

Multi-level approval works

CopilotKit UI works

Resume works

Agents work

MCP tools work

No breaking changes

Final Result After Phase 7

You now have enterprise-grade HITL:

Tool → approval workflow → role-based → multi-level → audited → executed

Fully configurable via YAML.
