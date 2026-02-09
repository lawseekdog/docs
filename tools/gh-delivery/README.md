# gh-delivery

Team helper scripts for the LawSeekDog multi-repo delivery workflow:

Issue (work item) -> PR -> GitHub Projects board.

## Prereqs

- GitHub CLI (`gh`)
- Token scopes include `project`:

```bash
gh auth refresh -s project,read:org,repo,workflow
gh auth status
```

## Quick start

```bash
./tools/gh-delivery/lsd-work doctor
./tools/gh-delivery/lsd-work claim lawseekdog/gateway-service#123 --agent Codex
./tools/gh-delivery/lsd-work move  lawseekdog/gateway-service#123 --status "In Review"
```

## What it does

- Adds the Issue to the org Project (default: `lawseekdog` project `#1`)
- Sets project fields:
  - `Status` (e.g. `Claimed`, `In Review`)
  - `Agent` (Codex/Cursor/Claude/Copilot/Human)
  - `Target Repo` (e.g. `lawseekdog/gateway-service`)
- Assigns the Issue to `@me` on `claim`
- Optionally syncs a status label on the Issue (`--sync-labels`)

## Config

Environment variables:

- `LSD_PROJECT_OWNER` (default: `lawseekdog`)
- `LSD_PROJECT_NUMBER` (default: `1`)
- `LSD_AGENT_DEFAULT` (default: `Codex`)

