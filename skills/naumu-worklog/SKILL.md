---
name: naumu-worklog
description: Team work log for Naumu. Use at the start of any work the team would benefit from knowing about - research, drafting, analysis, planning, outreach, or a single meaningful action - before the first search, tool call or draft; again on decisions and when done; and whenever something comes up the team should know later. Skip only what adds nothing to team context, like a quick one-line question.
---

# Naumu work log

Leave a short trail of the work in the team's Naumu space: open a thread when the work starts, follow up on decisions, post a summary when done. Also log anything a teammate would want to know later - a decision, a customer insight, a finding - even when it surfaces outside a task.

Log here:

Naumu work log: space <space slug> (<space id>), topic <topic name> (<topic id>).

Naumu fills that line in when the skill is downloaded from Settings -> Your agents. If it still shows placeholders, use the same line from the user's instructions or memory; if there is none, say once that the work log is not set up (Naumu -> Settings -> Your agents) and carry on without logging.

Stand down when a repo's `.naumu` or AGENTS.md owns logging, when you are replying inside a Naumu thread, or when the user says not to log.

## Loop

1. **Start** - before any other tool call: `naumu_delegate({ graphId: <space id>, topicIds: [<topic id>], task: "Starting work: <2-3 bullets>" })`. Keep the returned `threadId`.
2. **Milestones** - on a decision, pivot, blocker or finished deliverable: `naumu_delegate({ graphId, threadId, task: "<what changed and why>" })`.
3. **Done** - a bulleted summary into the same `threadId`:

```
Done: October newsletter draft
- Draft in Google Docs: <link>
- Lead story switched to the customer case study - stronger numbers
- Open: Ana to approve the subject line by Thursday
```

- Pass `topicIds` on the Start call only; later calls ignore it.
- Keep entries short: outcomes and reasons, no secrets or personal data.
- If a call says you are not a member of the space, call `naumu_resolve_admission({ graphId: <space id> })` once. If you joined, retry the entry. If it filed a join request, tell the user once that access was requested and to ask a space admin to approve it, then stop logging for this session.
- If any other call fails, say so in one line and keep working.
