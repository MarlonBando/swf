# swf design

How the factory works in practice. Decisions made so far, and what's still open.

## Pipeline

1. **Intake.** I brainstorm with Claude in chat (phone, PC or Claude Code). Claude creates the issue in the target repo.
2. **Worker.** When I say the ticket is ready, Claude adds the `swf:ready` label. That label is the only trigger. Creating an issue never starts work.
3. **Reviewer.** Runs right after the worker, with fresh context. It comments and suggests changes on the PR. It never pushes.
4. **Me.** I check the PR and merge. The PR closes the issue.

## Where it runs

**GitHub Actions**, using `anthropics/claude-code-action`. My laptop is mostly asleep, so it can't be the server.

- Compute runs on GitHub, billed to my GitHub account. It's free on public repos; private repos get 2,000 minutes a month on GitHub Free.
- Claude runs on my Claude Pro plan. Each repo stores a `CLAUDE_CODE_OAUTH_TOKEN` secret, created once with `claude setup-token`.
- The shared logic is a reusable workflow in `swf`. Each project has one small caller workflow.

## Ticket format

Every issue written by the brainstorm agent has three sections:

- **Goal**: what should be true when it's done.
- **Acceptance criteria**: checkable points.
- **Out of scope**: what not to touch.

## Status labels

```
swf:backlog → swf:ready → swf:in-progress → swf:in-review → merged
                               ↘ swf:needs-info ↗
```

| Label | Set by | Meaning |
|---|---|---|
| `swf:backlog` | Claude (chat) | Captured, not to be worked on yet |
| `swf:ready` | Claude, when I say so | **Trigger.** Worker starts |
| `swf:in-progress` | Worker | Being built |
| `swf:needs-info` | Worker | Questions posted on the issue. I answer, then set it back to `swf:ready` |
| `swf:in-review` | Worker | PR opened, reviewer has commented |

## Worker rules

- Work on a branch `swf/<issue-number>-<slug>` and open a PR that says `Closes #<n>`.
- **Always run the test suite before pushing.** If tests fail, don't open the PR: comment on the issue with the failure.
- If the ticket is ambiguous, comment questions on the issue, set `swf:needs-info`, and stop.

## Projects

| Repo | Visibility | Stack | Test command |
|---|---|---|---|
| [masbench](https://github.com/MarlonBando/masbench) | public | Go | `go test ./...` |
| [finalboss](https://github.com/MarlonBando/finalboss) | private | Go | `go test ./...` |
| [nextup](https://github.com/MarlonBando/nextup) | private | Go backend + static frontend | `cd backend && go test ./...` |

Adding a project means adding one caller workflow, one secret and the `swf:*` labels.

## Open

- Usage volume is unknown. Measure it after the first tickets, and see whether Pro limits are enough.
- Test commands above are a first guess. Confirm them for each repo.
