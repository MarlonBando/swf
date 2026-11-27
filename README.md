# swf: a personal software factory

I describe what I want, and the factory builds it.
The only interface is a Claude chat, on my phone or PC.

## The idea

- **Chat is the interface.** No dashboards, no terminal. I talk to Claude and that's it.
- **GitHub issues are the work.** Every change the factory should make is an issue in this repo.
- **Claude writes the issues.** Most come from our conversation. I can also open one myself at any time.

## How it works

1. **Idea.** I describe what I want in chat, in plain language.
2. **Issue.** Claude turns it into a GitHub issue with a clear scope.
3. **Build.** The factory works on the issue. *(Mechanism still to be decided.)*
4. **Report.** Progress and results come back to the chat, and the issue gets updated.

## Principles

- If it needs a screen to use, that's a problem to fix, not a feature.
- If it isn't in an issue, it isn't planned.
- Small issues, each one a single reviewable change.
- I stay in the loop: Claude proposes, I decide on anything that matters.

## Status

Early stage. This repo only holds this README.

## Open questions

- Who or what does the building, and where does it run?
- How do results come back: pull requests, issue comments, or both?
- What is the first project the factory should build?
