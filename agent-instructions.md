<!-- orbi:start -->
## Orbi

Orbi shows your work on the user's desktop through the local `orbi` MCP server.

- Report work when it helps the user follow it: several steps, minutes of work, or a wait. A quick answer or small edit needs no call. Use few calls and update only when something meaningful changes.
- Start each reported task with `introduce`: the repository or working folder's name, the task, `state: "working"`, and a `progress` widget for several steps. Its `data.value` is a number from 0 to 1; each step has `name` and `state` (`todo`, `running`, or `done`). A new task clears the previous task's widgets.
- Use `progress` for steps, `list` for checks, `stat` for a useful number, and `brief` for the result. Discover each tool's schema before using it. Batch widgets and removals with status updates when possible.
- Ask questions with `clarify`. If unanswered, stop dependent work and leave the question in your reply. Follow the agent's existing approval rules for actions that require permission.
- Finish reported work with one `set_status`: `finished` or `error`, a short message, a `brief` when useful, and outdated widget IDs in `remove`. Never leave a finished task showing as working.
- When subagents are authorized, report them with `subagent` before launch and when they finish. Do not launch subagents just because Orbi supports them. A subagent shares its parent's session and must not call Orbi tools except `subagent` for its own authorized children, using its ID as `parent`; it reports progress and questions to its parent.
- If Orbi is unavailable, continue the task and use chat for questions. Do not retry in a loop.
<!-- orbi:end -->
