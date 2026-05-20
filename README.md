# Skills

Personal collection of agent skills for Claude Code and other AI coding agents.

## Skills

| Skill | Description | Install |
|-------|-------------|---------|
| [npm-updater](skills/npm-updater) | Step-by-step dependency update workflow for Node.js projects. Handles minor/patch batching, major version analysis with changelog review, migration guide discovery, and post-update verification. Supports npm, yarn, pnpm, and bun. | `npx skills add verebelyin/skills --skill npm-updater` |
| [dependabot-pnpm-fix](skills/dependabot-pnpm-fix) | Batch-fix Dependabot vulnerability alerts in pnpm-based Node.js projects. Fetches open alerts via `gh api`, verifies patched versions on npm, and applies `pnpm.overrides` (or direct bumps) for direct and transitive dependencies. Includes a monorepo variant for single root lockfiles. | `npx skills add verebelyin/skills --skill dependabot-pnpm-fix` |

## Installation

Browse and discover skills at [skills.sh](https://skills.sh).

```bash
npx skills add verebelyin/skills
```
