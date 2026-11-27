# swf design

How the factory works in practice. Decisions made so far, and what's still open.

## Pipeline

1. **Intake.** I open a Claude Code session on the `swf` repo: the Code tab on my phone, or the CLI on my PC. With `/swf-ticket`, Claude brainstorms with me and files the ticket as an issue **in swf**, labeled `project:<repo>` and `swf:backlog`.
2. **Dispatch.** When I say the ticket is ready, Claude adds `swf:ready` to the swf issue. [`dispatch.yml`](../.github/workflows/dispatch.yml) creates the issue in the project repo with `swf:ready`, links the two, and closes the swf issue. The `swf:ready` label is the only trigger: creating an issue never starts work.
3. **Worker.** The project's `swf.yml` fires on `swf:ready` and runs the worker.
4. **Reviewer.** Runs right after the worker, with fresh context. It comments and suggests changes on the PR. It never pushes.
5. **Me.** I check the PR and merge. The PR closes the issue.

### Why tickets start in swf

A cloud session can only reach the repo it was started on. So I always start the session on `swf`, and the dispatcher, using the `SWF_DISPATCH_TOKEN` secret (a fine-grained token with Issues read/write on the project repos), carries the ticket over. swf issues are the intake log. Work and status live in the project repo.

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
| `swf:failed` | Worker | Couldn't finish (tests failing, crash). Reason on the issue. Re-add `swf:ready` to retry |

## Worker rules

- Work on a branch `swf/<issue-number>-<slug>` and open a PR that says `Closes #<n>`.
- **Always run the test suite before pushing.** If tests fail, don't open the PR: comment on the issue with the failure.
- If the ticket is ambiguous, comment questions on the issue, set `swf:needs-info`, and stop.

## Implementation

- [`factory.yml`](../.github/workflows/factory.yml) is the reusable workflow with two jobs: `worker`, then `reviewer`.
- Each project has `.github/workflows/swf.yml`, which fires on `issues: labeled` with `swf:ready` and passes the test command.
- The worker and reviewer run `anthropics/claude-code-action@v1` with the Claude GitHub App. PRs come from `claude[bot]`, so the project's own CI runs on them.
- **Tests before push are enforced**, not just requested: a git `pre-push` hook runs the test command, and `--no-verify` and force pushes are denied.
- The worker ends with structured output (`pr_opened` / `needs_info` / `failed`). A plain workflow step uses it to set labels, so a crash still ends as `swf:failed` with a link to the logs.
- The reviewer posts one comment-only review: a verdict, then findings with `path:line` and suggested changes. It can't edit, push, approve or merge.
- Turn caps (80 for the worker, 30 for the reviewer) and job timeouts limit usage of the Pro plan per ticket.

## Projects

| Repo | Visibility | Stack | Test command |
|---|---|---|---|
| [masbench](https://github.com/MarlonBando/masbench) | public | Go | `go test ./...` |
| [finalboss](https://github.com/MarlonBando/finalboss) | private | Go | `go test ./...` |
| [nextup](https://github.com/MarlonBando/nextup) | private | Go backend + static frontend | `cd backend && go test ./...` |
| [foosball-dtu](https://github.com/MarlonBando/foosball-dtu) | public | Go backend + React frontend | `cd backend && go test ./...` |

Adding a project means: one caller workflow, the `CLAUDE_CODE_OAUTH_TOKEN` secret and the `swf:*` labels in the project, plus a `project:<repo>` label in swf and the repo added to `SWF_DISPATCH_TOKEN`.

## Open

- Usage volume is unknown. Measure it after the first tickets, and see whether Pro limits are enough.
- Test commands above are a first guess. Confirm them for each repo.
- `masbench` fails `go test ./...` on `main` today (`go vet` flags `cmd/run.go:118`). That's the first ticket.
