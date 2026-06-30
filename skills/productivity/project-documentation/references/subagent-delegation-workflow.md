# Subagent Delegation Workflow

## When the User Says "Gunakan Subagent" or "Maksimalkan Progress"

The user wants parallel execution via `delegate_task` to speed up progress. This is a directive, not a suggestion.

## Pattern

```
1. Identify independent workstreams (3 max for this user)
2. Dispatch up to 3 subagents in parallel via delegate_task(tasks=[...])
3. While subagents work, continue on other tasks
4. Subagent results re-enter as a new message when all finish
5. Verify subagent output — subagents self-report, not verified facts
```

## Example (from FundManager V2 session)

```
Task 1: Build SupervisorInboxScreen + AssignTaskScreen
Task 2: Build ApprovalScreen + VerificationScreen
Main agent: Build LaporanPekerjaanScreen + navigation

→ 2 subagents dispatched, main agent continued work
→ All 4 screens completed in 157s (parallel), compiled together
→ APK Build #13 successful with all screens
```

## Pitfalls

- Subagents have NO context from the conversation. Pass ALL necessary info via `context` field: file paths, patterns to follow, existing conventions, forbidden imports.
- Subagents may use custom components that don't exist (e.g. `AppDropdownField`, `UiFormatters`). Explicitly list allowed/reference imports.
- Subagent results are self-reports — ALWAYS verify file existence and compile before telling the user.
- Max 3 concurrent subagents for this user (configured in config.yaml delegation.max_concurrent_children).
