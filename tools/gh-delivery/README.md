# gh-delivery

Deprecated location for the LawSeekDog multi-repo delivery workflow helper.

**Source of truth** lives in the org repo `lawseekdog/.github` as the `lsd-delivery` skill.

This folder keeps a tiny wrapper (`./tools/gh-delivery/lsd-work`) for backwards compatibility,
but it delegates to the installed skill script.

## Prereqs

- GitHub CLI (`gh`)
- Token scopes include `project`:

```bash
gh auth refresh -s project,read:org,repo,workflow
gh auth status
```

## Quick start

```bash
npx -y skills add lawseekdog/.github@lsd-delivery -g -y
~/.agents/skills/lsd-delivery/scripts/lsd-work bootstrap
~/.agents/skills/lsd-delivery/scripts/lsd-work doctor
```

Docs:

- See `/implementation/lsd-delivery.md` in this docs site.

## Config

Environment variables:

- `LSD_PROJECT_OWNER` (default: `lawseekdog`)
- `LSD_PROJECT_NUMBER` (default: `1`)
- `LSD_AGENT_DEFAULT` (default: `Codex`)
