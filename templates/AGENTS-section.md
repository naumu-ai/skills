## Naumu (MANDATORY)

The team's memory lives in the Naumu space named in `.naumu` (`space` is an identifier, not a secret). When a session's first message describes a build, fix, refactor or feature task:

- **Start.** Run one `naumu_search` on the task for prior context, then open the work-log thread: `naumu_delegate({ graphId: <.naumu space>, topicIds: [<.naumu tracking.topicId>], task: "Starting work: <2-3 bullets>" })`. Keep the returned `threadId` for the whole session. `topicIds` only works on this first call - a thread that misses it lands in `#misc` and can never be re-filed.
- **Milestones.** On a decision, a pivot, a blocker or a scope change, post a follow-up: `naumu_delegate({ graphId, threadId, task: "<what changed and why>" })`.
- **Done.** Post a bulleted summary of what changed into the same `threadId`, then add `Naumu-Thread: https://naumu.ai/spaces/<.naumu slug>/chat/<threadId>` as a trailer on every commit for the task.

Never block work on Naumu: if a call fails, say so in one line and continue. Keep entries short and bulleted. If no Naumu MCP server is connected, offer once to set it up and wait for a yes - never auto-run: `npx skills add naumu-ai/skills`, then `naumu_resolve_admission` with the space id.
