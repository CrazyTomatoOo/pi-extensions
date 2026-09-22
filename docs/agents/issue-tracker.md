# Issue tracker: GitHub

Issues and specs for this repo live in GitHub Issues at
`https://github.com/CrazyTomatoOo/pi-extensions`. Use the `gh` CLI for all
operations.

## Conventions

- **Create**: `gh issue create --title "..." --body "..."`.
- **Read**: `gh issue view <number> --comments`.
- **List**: `gh issue list --state open --json number,title,body,labels,comments`.
- **Comment**: `gh issue comment <number> --body "..."`.
- **Labels**: `gh issue edit <number> --add-label "..."` / `--remove-label "..."`.
- **Close**: `gh issue close <number> --comment "..."`.

Infer the repository from `git remote -v`; `gh` does this automatically inside
this checkout.

## Pull requests as a triage surface

**PRs as a request surface: no.** External pull requests are not included in
the issue triage queue by default.

## When a skill says "publish to the issue tracker"

Create a GitHub issue.

## When a skill says "fetch the relevant ticket"

Run `gh issue view <number> --comments`.

## Wayfinding operations

Used by `/wayfinder`. The map is a single GitHub issue with child issues as
tickets.

- **Map**: one issue labelled `wayfinder:map`, holding Notes,
  Decisions-so-far, and Fog.
- **Child ticket**: a GitHub sub-issue labelled
  `wayfinder:research`, `wayfinder:prototype`, `wayfinder:grilling`, or
  `wayfinder:task`. If sub-issues are unavailable, put `Part of #<map>` at the
  top of the child body.
- **Blocking**: use GitHub native issue dependencies where available. Otherwise
  record `Blocked by: #<n>, #<n>` in the child body.
- **Frontier**: open, unblocked, unassigned child issues, in map order.
- **Claim**: assign the ticket to the driving developer before work begins.
- **Resolve**: comment the answer, close the issue, then append a context
  pointer to the map's Decisions-so-far.
