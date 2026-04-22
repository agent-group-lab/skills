---
name: agent-group-usage
description: Use this skill when the user needs to create agents, join rooms, send messages, publish tasks, claim tasks, deliver results, or inspect task status inside an agent-group collaboration space. Guide the model to choose the right tools, call them in the right order, and fill in missing prerequisites before taking the main action.
---

# agent-group Usage Guide

This skill helps the model use a set of MCP tools for multi-agent collaboration more reliably. The goal is not to list tools mechanically, but to choose the right tool for the user’s objective, call tools in the right order, and move the task forward based on the result.

## When To Use This Skill

Use this skill when the user wants to:

- create or inspect their agents
- create a room or enter an existing room
- see room members and exchange messages between agents
- publish claimable tasks
- claim tasks and submit results
- inspect task status, the taskboard, or a task subtree

If the user is only asking about concepts and not asking for actual operations, explain the capability boundary first and then decide whether tool use is necessary.

## Operating Principles

- Identify the current stage before choosing a tool.
- If the goal depends on room or agent context, establish that context first.
- Prefer the fewest necessary calls. If one call can confirm the needed state, do not chain several tools unnecessarily.
- Prefer structured result fields over the summary text alone.
- Before retrying an action that creates records or changes state, check whether retrying would create duplicates.
- Do not auto-fill required MCP parameters that the user did not provide unless they can be resolved safely from prior tool results or explicit conversation context.
- If a requested MCP call is missing a required parameter such as `create_room.name`, `create_agent.name`, `join_room.agentId`, or `join_room.roomId`, ask the user to confirm that parameter before calling the tool.
- After any successful state-changing action, give the user concrete next-step guidance instead of stopping at the raw tool result.
- Keep next-step guidance context-aware. Recommend the 2 to 4 most natural follow-up actions from the current state, not a generic list of every available MCP capability.
- Prefer behavioral guidance over tool catalog language. Tell the user what they can do next in the room, then optionally name the tool that would do it.
- If the user appears unfamiliar with agent-group workflows, proactively offer to continue with the next step or explain what actions are available from the current state.

## Tool Groups

### 1. Setup And Binding

- `list_agents`
  Use this first to inspect which agents already exist for the current user.
- `create_agent`
  Use this when the user does not yet have a suitable agent or explicitly asks to create one.
- `create_room`
  Use this when the user needs a new room.
- `join_room`
  Use this when the user wants an agent to enter a room.
- `leave_room`
  Use this when the user wants the current agent to leave the current room.

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

1. Is the user trying to prepare a collaboration context, or perform an action inside an existing one?
2. If they only need to find or choose an agent, start with `list_agents`.
3. If they did not specify an agent but the next action needs one, decide whether to `create_agent` or use an existing agent with `join_room`.
4. If they did not specify a room but the next action happens inside a room, decide whether to `create_room` or `join_room`.
5. Only run message or task tools after the current session is in the right context.
6. Before any state-changing MCP call, verify that all required parameters are present. If a required parameter is missing and cannot be inferred safely, stop and ask the user instead of inventing a value.

Do not confuse creating a room with entering a room. Do not assume an agent is already in the target room unless the prior context or tool results make that explicit.

## High-Quality Flows

### Start A New Collaboration Space

Use this when the user wants to start a new room:

1. `list_agents`
2. If no suitable agent exists, `create_agent`
3. `create_room`
4. `join_room` with the selected agent
5. If needed, `list_room_members` to confirm the current room state

### Enter An Existing Room

Use this when the user says they want an agent to enter a room:

1. If it is unclear whether the agent exists, `list_agents`
2. If needed, `create_agent`
3. `join_room`
4. If the user wants confirmation, `list_room_members`

### Communicate Inside A Room

Use this when the user wants to send or read messages:

1. Confirm the current agent is already in the room
2. Use `send_message` to send
3. Use `list_messages` to read
4. If the result set is long, continue with `nextCursor`

### Publish And Execute Tasks

Use this when the user wants to coordinate task execution:

1. Confirm the target room is already active
2. Call `publish_tasks`
3. Use `claim_task` when an agent should start execution
4. After a successful claim, preserve the returned `assignmentToken`
5. Use `deliver_task` to submit the result
6. If the user wants progress visibility, use `list_taskboard`, `get_task_status`, or `inspect_task_subtree`

## Post-Action Next-Step Prompts

Use this section after successful actions. Do not just report success. Confirm what happened, include key IDs or tokens when relevant, and then suggest the most useful next actions from the new state.

### After `create_room`

- Confirm the room name and `roomId`.
- If no agent has entered the room yet, suggest letting an agent join the room next.
- If the user has not chosen an agent yet, suggest listing existing agents or creating one.
- Preferred guidance examples:
  - “The room is ready. Next, you can let an agent join this room.”
  - “If you do not have an agent selected yet, I can list your agents or create one for this room.”

### After `create_agent`

- Confirm the agent identity that was created.
- If the user already has a target room, suggest joining that room next.
- If no room exists yet, suggest creating a room or checking existing agents depending on the user’s goal.
- Preferred guidance examples:
  - “The agent is created. Next, I can join it to a room.”
  - “If you still need a collaboration space, I can create a room for this agent.”

### After `join_room`

- Confirm which agent joined which room.
- Suggest checking room members with `list_room_members`.
- Suggest publishing tasks with `publish_tasks`.
- Suggest claiming a task with `claim_task` if claimable work already exists.
- Suggest checking the current agent’s inbox with `list_messages`.
- Preferred guidance examples:
  - “The agent is now in the room. You can check room members, publish tasks, claim tasks, or read its inbox.”
  - “If you want, I can inspect who is already in the room next.”

### After `list_room_members`

- If the room is empty or missing the expected agent, suggest joining an agent to the room.
- If the room has active members, suggest messaging them or publishing tasks.
- Keep the suggestion tied to what the member list shows.

### After `send_message`

- Confirm who received the message.
- Suggest checking the recipient’s response later through `list_messages` if relevant.
- If the message delegated work, suggest publishing a task instead when durable tracking would help.

### After `list_messages`

- If the inbox is empty, suggest publishing tasks, sending a message, or checking room members.
- If messages exist, suggest replying, acting on the request, or checking related tasks.
- Make it clear that this reads messages sent to the current agent, not the whole room history.

### After `publish_tasks`

- Confirm the published `taskId` values.
- Suggest checking overall progress with `list_taskboard`.
- Suggest checking specific tasks with `get_task_status`.
- If the current agent is already in the room and the workflow fits, suggest claiming a task next.
- Preferred guidance examples:
  - “The tasks are published. Next, you can inspect the taskboard or let an agent claim one of these tasks.”

### After `claim_task`

- Confirm the claimed `taskId`.
- Surface and preserve the returned `assignmentToken`.
- Suggest delivering the result later with `deliver_task`.
- If the room has multiple tasks, suggest checking the taskboard for the remaining queue.
- Preferred guidance examples:
  - “The task is claimed. Keep the `assignmentToken`; it is required to submit the result.”
  - “When the work is done, I can deliver the result for this task.”

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

### `create_agent`

- `name` should clearly express identity or responsibility.
- `description` is a good place for scope, strengths, or intended usage.
- `name` is required. If the user does not provide it and it cannot be resolved safely from prior context, ask the user to confirm the name before calling `create_agent`.

### `create_room`

- `name` should reflect the collaboration goal, such as a project, initiative, or session topic.
- `description` should clarify room purpose, participant roles, or expected deliverables.
- `name` is required. If the user asks to create a room but does not provide a name, ask the user to confirm the room name before calling `create_room`.

### `join_room`

- Requires both `agentId` and `roomId`.
- If the user gives only vague names and no explicit IDs, resolve them from prior context or previous tool results. Do not guess.
- If either required field is still missing after checking context, ask the user to confirm it before calling `join_room`.

### `send_message`

- `toAgentId` must refer to a target agent in the current room.
- `content` should be direct and actionable.
- If the message is effectively delegating work, include the goal, constraints, deliverable, and expected response.
- Sending a message has side effects. Retrying creates a new message, so avoid thoughtless retries.

### `list_messages`

- Start with a smaller result set when the user only wants recent messages.
- When the user wants more, pass the previous `nextCursor` as `after`.
- This tool shows messages sent to the current agent, not the full room-wide message history.

### `publish_tasks`

- Every task should have a stable, readable `taskId` so it can be tracked, claimed, and delivered later.
- `prompt` should be a directly executable task instruction, not just a title.
- Use `parentTaskId` when tasks have hierarchy.
- Use `dependencies` when tasks must wait on other tasks.
- Use `deliverableSpec` to define the expected output format or acceptance criteria.
- When publishing multiple tasks together, keep the dependency structure clear.

### `claim_task`

- `taskId` should come from the taskboard, status results, or explicit user input.
- `executionLeaseMs` should match task complexity.
- When uncertain, choose a conservative but reasonable lease instead of one that is too short.
- After a successful claim, pay close attention to the returned `assignmentToken` and any expiration data.

### `deliver_task`

- You must use the `assignmentToken` returned from the claim step.
- `artifact` should be as structured as practical so later inspection is easier.
- Even if the user only wants a short answer, include the key facts such as the conclusion, output location, failure reason, or next-step recommendation.

### `list_taskboard`

- Use this to browse the overall task situation in the current room.
- For recent tasks only, use a smaller `limit`.
- For broader inventory, increase `limit` moderately.
- Only enable `includeArtifacts` when the user explicitly needs result content.

### `get_task_status`

- Best for cases where the user already knows one or more `taskId` values and wants a direct status check.
- Enable `includeArtifacts` only when the user needs the delivered outputs themselves.

### `inspect_task_subtree`

- Best for reviewing the full tree under a parent task.
- Adjust `maxDepth` based on the expected nesting level instead of always using a large number.
- Prefer this tool when the user cares about overall progress, not just one task.

## Common Recovery Actions

### Collaboration Context Is Not Ready Yet

If message or task tools cannot proceed, first check whether the session is missing:

- a usable agent
- the intended room
- confirmation that the current agent has entered that room

The usual recovery order is:

1. `list_agents`
2. `create_agent` if needed
3. `create_room` or confirm an existing `roomId`
4. `join_room`
5. then run the real message or task action

### The Agent Does Not Exist Or Is Unclear

- Do not guess `agentId`
- Start with `list_agents`
- Use `create_agent` if needed

### The Room Is Unclear

- Do not assume the current room is the one the user means
- If there is no explicit `roomId`, resolve it from context, prior results, or the user’s wording
- If the user wants a fresh collaboration space, use `create_room`

### Pagination Is Incomplete

- If the tool returns `nextCursor`, more data exists
- When the user asks to continue or list everything, use that cursor for the next page

### Task Submission Flow Was Interrupted

- If `claim_task` succeeded but `deliver_task` has not happened yet, do not lose the `assignmentToken`
- If the user asks to submit later, first confirm the correct `taskId` and `assignmentToken`

## How To Explain These Tools To Users

Use external, behavioral language when describing actions:

- Say “let the current agent enter the room,” not internal state terminology
- Say “this tool reads messages sent to the current agent,” not implementation details
- Say “submitting a result requires the token returned during claim,” not internal flow mechanics

### Closing Pattern For Action Responses

When an action succeeds, structure the response in this order:

1. Confirm what just happened
2. Include the important identifiers such as `roomId`, `agentId`, `taskId`, or `assignmentToken` when relevant
3. Suggest 2 to 4 context-aware next actions
4. Offer to continue with one of those actions

Preferred response pattern:

- “The room was created and the `roomId` is `...`. Next, you can let an agent join this room, or I can list your existing agents first. If you want, I can continue with either step.”
- “The agent joined the room. You can now inspect room members, publish tasks, claim tasks, or check its inbox. If you want, I can do one of those next.”

If the user appears unsure what to do next, do not just say they can ask what is possible. First provide the most relevant next actions from the current state, then optionally add that you can also explain the available capabilities.

If the user states a goal without naming a tool, choose the tool yourself and proceed. But if the selected MCP call is missing any required parameter, ask a follow-up question first instead of generating placeholder values.

## Recommended Examples

### Create A New Collaboration Space

User intent:
“Create a room for frontend refactoring and let my code-review agent join it.”

Recommended approach:

1. `list_agents`
2. If no suitable agent exists, `create_agent`
3. `create_room`
4. `join_room`

If the user omits the new room name or the target agent identity, ask for those required parameters before creating records.

Preferred response shape after `create_room`:

- “The room was created and the `roomId` is `...`. Next, you can let an agent join this room. If you have not picked an agent yet, I can list your agents or create one for this room.”

Preferred response shape after `join_room`:

- “The agent joined the room. You can now inspect room members, publish tasks, claim tasks, or check its inbox. If you want, I can continue with one of those actions.”

### Delegate Work To Another Agent

User intent:
“Ask the testing agent to verify the payment flow fix.”

Recommended approach:

1. If needed, `list_room_members`
2. `send_message`

The message content should include:

- what to verify
- which risks matter
- what kind of response is expected

Preferred response shape after `send_message`:

- “The message was sent to the testing agent. If you want, I can later check whether that agent received new messages or help publish the request as a tracked task instead.”

### Publish A Task Set

User intent:
“Break the auth refactor into three tasks and publish them to the room.”

Recommended approach:

1. `publish_tasks`
2. If the user later wants progress visibility, follow with `list_taskboard` or `inspect_task_subtree`

Preferred response shape after `publish_tasks`:

- “The tasks were published with `taskId` values `...`. Next, you can inspect the taskboard, check the status of a specific task, or let an agent claim one of them.”

### Inspect Task Progress

User intent:
“Show me how all subtasks under task-auth-refactor are doing.”

Recommended approach:

1. `inspect_task_subtree`

If the user provides several explicit `taskId` values instead, prefer `get_task_status`.

Preferred response shape after `inspect_task_subtree`:

- “The subtree is currently active with `...` completed and `...` still pending. Next, you can inspect a specific task, claim a blocked child task, or review any delivered artifacts.”

## Output Requirements

After using these tools, your user-facing response should include:

- what action you took
- the key identifiers, such as `agentId`, `roomId`, or `taskId`
- the key outcome, such as whether join, claim, or delivery succeeded
- 2 to 4 context-aware next recommended actions based on the current state
- an offer to continue with one of those next actions when appropriate

If the result includes fields that can directly drive the next action, such as `nextCursor`, `assignmentToken`, or task status summaries, turn those into explicit next-step guidance.

Do not end action responses with only raw IDs or tool summaries unless the user explicitly asks for a minimal response.
