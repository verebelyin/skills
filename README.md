# Skills

Personal collection of agent skills for Claude Code and other AI coding agents.

## Skills

| Skill | Description | Install |
|-------|-------------|---------|
| [npm-updater](skills/npm-updater) | Step-by-step dependency update workflow for Node.js projects. Handles minor/patch batching, major version analysis with changelog review, migration guide discovery, and post-update verification. Supports npm, yarn, pnpm, and bun. | `npx skills add verebelyin/skills --skill npm-updater` |
| [dependabot-pnpm-fix](skills/dependabot-pnpm-fix) | Batch-fix Dependabot vulnerability alerts in pnpm-based Node.js projects. Fetches open alerts via `gh api`, verifies patched versions on npm, and applies `pnpm.overrides` (or direct bumps) for direct and transitive dependencies. Includes a monorepo variant for single root lockfiles. | `npx skills add verebelyin/skills --skill dependabot-pnpm-fix` |
| [html-output](skills/html-output) | Produce a single, self-contained HTML file for rich output instead of a long Markdown reply — plans, side-by-side comparisons, code reviews, research explainers, interactive prototypes, and custom editors. Tailwind (and optional Mermaid) from CDN, round-trip "copy as Markdown/JSON" exports, and clean diagram conventions; writes to a temp file and opens it. | `npx skills add verebelyin/skills --skill html-output` |
| [create-pr](skills/create-pr) | Ship completed changes as a merged-ready pull request. Validates the working tree, creates a conventional branch and commit, opens the PR, requests Copilot review, and follows CI and review feedback through completion. | `npx skills add verebelyin/skills --skill create-pr` |

## Installation

Browse and discover skills at [skills.sh](https://skills.sh).

```bash
npx skills add verebelyin/skills
```
