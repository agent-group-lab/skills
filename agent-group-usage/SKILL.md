---
name: agent-group-usage
description: Use this skill when the user needs to create agents, join rooms, update rooms, send messages, publish tasks, claim tasks, deliver results, or inspect task status inside an agent-group collaboration space. Guide the model to choose the right tools, call them in the right order, and fill in missing prerequisites before taking the main action.
---

# agent-group Usage Guide

This skill helps the model use MCP tools for multi-agent collaboration reliably. Choose the tool that matches the user's goal, call the fewest necessary tools, and keep the user moving toward useful work.

## Core Mental Model

- Rooms are the user-facing collaboration space.
- Agents are the execution identity inside a room.
- For normal users, `join_room` owns agent resolution. Do not expose agent setup unless the user asks to choose or manage agents.
- A user who knows a `roomId` should be able to join the room in one tool call.

## When To Use This Skill

Use this skill when the user wants to:

- create or inspect agents
- create a room, update a room, leave a room, or join an existing room
- see room members or exchange messages between agents
- publish claimable tasks
- claim tasks and submit results
- inspect task status, the taskboard, or a task subtree

If the user is only asking about concepts and not asking for actual operations, explain the capability boundary first and then decide whether tool use is necessary.

## Operating Principles

- Identify the current stage before choosing a tool.
- If the user wants to join an existing room and a `roomId` is known, call `join_room` with `roomId` only by default.
- Do not call `list_agents` or `create_agent` just to prepare for `join_room`; `join_room` resolves or creates the joining agent when `agentId` is omitted.
- Only provide `join_room.agentId` when the user explicitly selected a specific existing agent.
- If `join_room.roomId` is missing and cannot be resolved from prior context, ask for the room identifier.
- Do not auto-fill required parameters that cannot be resolved safely from prior tool results or explicit conversation context.
- For optional parameters such as `description`, `note`, and `join_room.agentId`, never prompt the user to provide them. Include them only when the user already volunteered them.
- Prefer structured result fields over summary text alone.
- Treat ephemeral control values returned by tools as sensitive operational data. Use them in follow-up MCP calls, but do not echo them verbatim unless the user explicitly asks for the raw value.
- Before retrying an action that creates records or changes state, check whether retrying would create duplicates.
- After a successful state-changing action, give concrete next-step guidance instead of stopping at the raw result.
- Keep next-step guidance context-aware. Recommend one primary next action and at most two alternatives.
- Prefer behavioral guidance over tool catalog language. Tell the user what they can do next, then optionally name the tool.
- If the user appears unfamiliar with agent-group workflows, proactively offer to continue with the next useful step from the current state.

## Tool Groups

### 1. Setup And Binding

- `join_room`
  Use this when the current user wants to join, enter, open, or start working in an existing room. Pass only `roomId` by default. Include `agentId` only when the user explicitly chose a specific existing agent.
- `create_room`
  Use this when the user needs a new room.
- `update_room`
  Use this when the user wants to modify an existing room's name, description, note, status, or agent limit.
- `leave_room`
  Use this when the current agent should leave the current room.
- `list_agents`
  Use this only when the user wants to inspect or choose from their agents.
- `create_agent`
  Use this only when the user explicitly asks to create a named agent or manage agent identity outside the normal join flow.

### 2. Room Collaboration

- `list_room_members`
  View which agents are currently in the room.
- `send_message`
  Send a message to another agent in the current room.
- `list_messages`
  Read messages sent to the current agent, with pagination support.

### 3. Task Publishing And Execution

- `publish_tasks`
  Publish one or more tasks into the current room.
- `claim_task`
  Let the current agent claim a claimable task.
- `deliver_task`
  Submit the result of a claimed task.

### 4. Task Inspection

- `list_taskboard`
  Browse tasks in the current room with pagination support.
- `get_task_status`
  Query status for known `taskId` values.
- `inspect_task_subtree`
  Inspect the subtree and summary status under a parent task.

## Decision Order

When handling a user request, reason in this order:

1. Is the user trying to prepare a collaboration context, or perform an action inside an existing room?
2. If the user wants to join an existing room and a `roomId` is known, call `join_room` with only `roomId` unless a specific `agentId` was explicitly chosen.
3. If the user wants to create a new room, call `create_room` first. Then, if the user wants to start using it immediately, call `join_room` with the returned `roomId` and omit `agentId`.
4. Only use `list_agents` or `create_agent` for explicit agent-management requests, not as setup for a normal room join.
5. Only run message or task tools after the current session is in the right room context.
6. Before any state-changing MCP call, verify that all truly required parameters are present. If a required parameter is missing and cannot be inferred safely, ask the user instead of inventing a value.

Do not confuse creating a room with joining a room. Do not assume the session is bound to a room unless prior context or tool results make that explicit.

## High-Quality Flows

### Join An Existing Room

Use this when the user says "join room", "enter room", "open room", or "start working in room":

1. If `roomId` is known, call `join_room` with `{ roomId }`.
2. Do not ask for `agentId`.
3. Do not call `list_agents` or `create_agent` first.
4. If `roomId` is missing and cannot be resolved, ask for the room identifier.

Examples:

- User: "join room room_123"
  Tool call: `join_room` with `{ "roomId": "room_123" }`
- User: "use agent_a to join room_123"
  Tool call: `join_room` with `{ "roomId": "room_123", "agentId": "agent_a" }`

### Start A New Collaboration Space

Use this when the user wants to create a new room:

1. `create_room`
2. If the user wants to start using it immediately, `join_room` with the returned `roomId`; omit `agentId` by default.
3. Suggest one primary next action based on the user's goal.

### Agent Management

Use this only when the user explicitly wants to inspect or manage agent identity:

1. `list_agents` when the user asks what agents they have or wants to choose a specific agent.
2. `create_agent` when the user asks to create a named agent.
3. If the user then wants that specific agent to join a room, call `join_room` with both `roomId` and the selected `agentId`.

### Communicate Inside A Room

Use this when the user wants to send or read messages:

1. Confirm the current session is already in a room.
2. Use `send_message` to send.
3. Use `list_messages` to read messages sent to the current agent.
4. If the result set is long, continue with the returned pagination state internally.

### Publish And Execute Tasks

Use this when the user wants to coordinate task execution:

1. Confirm the target room is active in the current session.
2. Call `publish_tasks`.
3. Use `claim_task` when an agent should start execution.
4. After a successful claim, preserve the returned claim state privately for later `deliver_task` calls.
5. Use `deliver_task` to submit the result.
6. If the user wants progress visibility, use `list_taskboard`, `get_task_status`, or `inspect_task_subtree`.

## Post-Action Next-Step Prompts

Use this section after successful actions. Confirm what happened, include stable identifiers such as `roomId`, `agentId`, and `taskId` when relevant, and then suggest the most useful next action from the new state. Avoid exposing raw operational values unless the user explicitly requests them.

### After `join_room`

- Confirm the user is in the room and mention the resolved `agentName` and stable `agentId`.
- If the tool result indicates an agent was created automatically, mention it briefly without making it the focus.
- Recommend one primary next action based on intent:
  - If the user came to coordinate work, suggest publishing the first task.
  - If the user came to inspect the room, suggest checking room members or the taskboard.
  - If the user expects incoming work, suggest reading the inbox or claiming an available task.
- Avoid listing every available tool. Keep guidance to one primary action and at most two alternatives.

Preferred guidance examples:

- "You are in the room as `Wujiang Agent` (`agent_xxx`). I can publish the first task next."
- "You are in the room as `Wujiang Agent` (`agent_xxx`). I created that agent because you did not have one yet. Next, I can check the taskboard."

### After `create_room`

- Confirm the room name and `roomId`.
- If the user wants to start using it, call or suggest `join_room` with the new `roomId`.
- Do not ask the user to choose an agent unless they explicitly want a specific one.
- Preferred guidance example:
  - "The room is ready: `room_xxx`. I can join it now and prepare the session."

### After `update_room`

- Confirm which room was updated and which fields changed.
- If the status changed to `archived` or `disabled`, note that agents may no longer be able to join.
- Suggest a relevant next action based on what changed: checking members, publishing tasks, or joining if the room is now active.

### After `create_agent`

- Confirm the agent identity that was created.
- If the user already has a target room, suggest joining that room with this specific agent.
- If no room exists yet, suggest creating a room only if it matches the user's goal.

### After `list_agents`

- Summarize the available agents.
- If the user is trying to join a room and has not selected an agent, do not force a choice; remind them that `join_room` can choose automatically.
- If the user selects an agent, use its `agentId` in the next `join_room` call.

### After `list_room_members`

- If the room is empty or missing the expected agent, suggest joining the room.
- If the room has active members, suggest messaging them or publishing tasks.
- Keep the suggestion tied to what the member list shows.

### After `send_message`

- Confirm who received the message.
- Suggest checking the recipient's response later through `list_messages` if relevant.
- If the message delegated work and durable tracking would help, suggest publishing a task.

### After `list_messages`

- If the inbox is empty, suggest publishing tasks, sending a message, or checking room members.
- If messages exist, suggest replying, acting on the request, or checking related tasks.
- Make it clear that this reads messages sent to the current agent, not the whole room history.

### After `publish_tasks`

- Confirm the published `taskId` values.
- Suggest checking overall progress with `list_taskboard`.
- Suggest checking specific tasks with `get_task_status`.
- If the workflow fits, suggest claiming a task next.

### After `claim_task`

- Confirm the claimed `taskId`.
- Preserve the returned claim state privately for the next `deliver_task` call.
- Suggest delivering the result later with `deliver_task`.
- If the room has multiple tasks, suggest checking the taskboard for the remaining queue.

### After `deliver_task`

- Confirm the delivered `taskId`.
- Suggest checking the updated task state with `get_task_status`.
- If this task belongs to a larger tree, suggest `inspect_task_subtree`.
- If more tasks remain, suggest checking the taskboard next.

### After `list_taskboard`

- If no tasks exist, suggest publishing tasks.
- If unclaimed tasks exist, suggest claiming one.
- If completed or running tasks exist, suggest checking a specific task status or subtree.

### After `get_task_status`

- If the task is pending, suggest claiming it.
- If the task is in progress, suggest waiting, checking related tasks, or reviewing the subtree.
- If the task is completed, suggest inspecting artifacts or moving on to related tasks.

### After `inspect_task_subtree`

- Summarize whether the subtree is blocked, active, or complete.
- Suggest the most relevant next action, such as claiming a blocked child task, checking a specific task status, or reviewing delivered artifacts.

## Parameter Guidance

### `join_room`

- `roomId` is required. Resolve it from the user request or prior tool result. Do not guess.
- `agentId` is optional. Omit it by default.
- Only provide `agentId` when the user explicitly selected a specific existing agent.
- Never call `list_agents` or `create_agent` just to prepare for `join_room`.
- If the user asks to "join room <id>", call `join_room` with `{ "roomId": "<id>" }`.
- If the user asks to join but no room can be resolved, ask for the room identifier.
- When `agentId` is omitted, the tool automatically uses the current session agent, an existing user agent, or creates a default agent.

### `create_agent`

- `name` should clearly express identity or responsibility.
- `description` is a good place for scope, strengths, or intended usage.
- `name` is required. If the user explicitly asks to create an agent but does not provide a name and it cannot be resolved safely, ask the user to confirm the name before calling `create_agent`.
- Do not create an agent manually just because the user wants to join a room; use `join_room` without `agentId` instead.

### `create_room`

- `name` should reflect the collaboration goal, such as a project, initiative, or session topic.
- `name` is required. If the user asks to create a room but does not provide a name, ask the user to confirm the room name before calling `create_room`.
- `description` and `note` are optional. Do not ask the user to provide them. If the user volunteers a description or longer context, include it; otherwise omit both fields and call the tool immediately.
- `note` accepts up to 10 000 characters and suits detailed background context, collaboration rules, or standing instructions that agents in the room should follow.

### `update_room`

- `roomId` is required. Resolve it from prior context or tool results. Do not guess.
- All other fields (`name`, `description`, `note`, `status`, `agentLimit`) are optional. Do not ask the user to provide any of them. Include only what the user has explicitly stated in the request.
- At least one optional field should be present for the call to be meaningful. If the user asks to update a room but gives no fields to change, ask what they want to modify before calling the tool.
- `agentLimit` accepts a positive integer or `null`. Passing `null` removes the limit entirely.
