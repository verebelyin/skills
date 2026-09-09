---
name: create-pr
description: 'Use when the code is already changed and the user asks to "create a PR", "open a pull request", "ship this", "push and PR this", "commit and push", or "raise a PR for these changes".'
argument-hint: 'Optional: target base branch, PR title, or scope hint'
---

# Create a PR

**Skill type: Goal skill.** The end-state is one concrete thing: a merged-ready, non-draft PR whose CI checks pass and whose Copilot review comments are all triaged (fixed or explicitly dismissed as false positives). This skill covers the *ship* half only — the code changes already exist. Do not implement new features here; if the change isn't written yet, implement it first.

## Procedure

### 1. Validate the working tree

- `git status --short` and `git diff --stat` to see exactly what will be shipped. Confirm nothing unrelated (build output, local scratch files, secrets, `.env`) is staged or untracked-but-about-to-be-added.
- Detect the toolchain from the repo (`package.json` scripts, `Taskfile.yml`, `pom.xml`, `Makefile`, `*.tf`) and run whatever exists, in this order:
  1. build (`pnpm build`, `npm run build`, `mvn -B package`, `go build ./...`)
  2. tests (`pnpm test`, `mvn -B test`, `pytest`)
  3. lint / format (`pnpm lint`, `terraform fmt -recursive`, `ruff check`)
- **Stop and fix failures before continuing.** Do not create a branch on top of a red build.
- If a check genuinely cannot run in this environment (needs network, credentials, a device), say so explicitly in the final report instead of silently skipping it.

### 2. Branch

- Determine the base branch: `git remote show origin | grep 'HEAD branch'`, or prefer `develop` if it exists and the repo's recent PRs target it.
- Branch from an up-to-date base:
  ```bash
  BASE=main
  BRANCH=feat/short-kebab-description
  git fetch origin
  git switch -c "$BRANCH" "origin/$BASE"
  ```
- Prefix with the semantic-release type matching the change: `feat/`, `fix/`, `chore/`, `docs/`, `refactor/`, `test/`, `ci/`, `perf/`, `build/`.
- Keep the suffix short, lowercase, kebab-case, and descriptive of the outcome (`fix/null-check-on-schema-load`, not `fix/patch1`).

### 3. Commit

Follow [Conventional Commits](https://www.conventionalcommits.org/).

**Header** — `<type>(<optional-scope>): <what was done>`. State the outcome and its value: what it enables, what bug it removes. Imperative mood, lowercase after the colon, no trailing period, under ~72 chars.

**Body** — a blank line, then a short bullet list of *what changed and why*. One bullet per distinct concern. No restating the header.

Always write the message with a heredoc so real newlines survive:

```bash
git add path/to/changed-file path/to/another-file
git commit -F - <<'EOF'
fix(schema-registry): reject malformed payloads before enrichment

- Validate against the registered JSON schema on ingest so bad events fail fast instead of corrupting downstream state
- Return a structured error with the failing field path to make triage possible
- Cover the empty-payload and unknown-version cases with tests
EOF
```

Then verify: `git log -1 --format=%B | sed -n l` — confirm no literal `\n` reached the message.

**Never include** any reference to an AI assistant, LLM, model name, "generated with", or `Co-Authored-By` for a bot in the git commit message, branch name, PR title, or normal PR prose. The single permitted exception is the final Model information line specified below.

### 4. Push and open the PR

```bash
git push -u origin "$BRANCH"
gh pr create --base "$BASE" --title "$TITLE" --body-file "$PR_BODY"
```

- **Not a draft.** Never pass `--draft`.
- **Title**: the Conventional Commit header, plain language, no jargon.
- **Body**: write the complete body to a fresh file (for example, a unique `mktemp` path) or explicitly truncate an existing file before passing `--body-file`; never append to a reused temporary path. Use exactly one `## Summary` and one `## Changes` section. Structure:
  - **Summary** — the problem in one or two sentences, then how it was fixed.
  - **Changes** — concise bullets of the concrete edits; group related config and lockfile updates when they represent one concern.
  - **Scope** — optional; include it only when it adds a meaningful boundary or exclusion that is not already clear from the summary and changes.
  - **Background** — optional; include it only when an issue, ticket, or trigger adds useful context.
- Before creating the PR, inspect the body file and confirm it contains no stale text or duplicated top-level sections.
- **Do not add a validation/testing-steps section.**
- **Model information** — make the final line of the PR body exactly one model-information line, after all sections. Before the review runs, use `effort=unverified`; replace it with `effort=Lite (verified)` only after the Copilot overview confirms Lite:
  ```text
  Model information: model=GitHub-managed; reviewer=GitHub Copilot Code Review; effort=unverified
  ```
  Use the actual model slug only when this workflow knows it. Copilot Code Review does not expose its exact model through the reviewer API, so use `model=GitHub-managed` for that path. This line is allowed in the PR body only. Keep model information out of the PR title, branch name, and every git commit message.

  After Lite is confirmed, rewrite the final line in the body file and update the PR:
  ```bash
  gh pr edit "$PR_NUMBER" --repo "$OWNER/$REPO" --body-file "$PR_BODY"
  ```

### 5. Request Copilot review

- Request Copilot through GitHub's documented reviewer API. The request body has no model or review-effort field:
  ```bash
  OWNER=org
  REPO=repo
  PR_NUMBER=123
  gh api --method POST \
    -H 'Accept: application/vnd.github+json' \
    -H 'X-GitHub-Api-Version: 2026-03-10' \
    "repos/$OWNER/$REPO/pulls/$PR_NUMBER/requested_reviewers" \
    --input - > /tmp/copilot-review-request.txt 2>&1 <<'JSON'
{"reviewers":["copilot-pull-request-reviewer[bot]"],"team_reviewers":[]}
JSON
  cat /tmp/copilot-review-request.txt
  ```
- The intended review effort for this skill is **Lite**. Configure the repository or organization default to `Lite` in GitHub Copilot code-review settings before running this workflow. The reviewer API cannot select `Lite`; never add undocumented `model`, `review_effort`, or prompt fields to the request.
- Treat `Lite` as verified only when the Copilot overview comment for this review reports `Lite`. If the overview reports `Balanced`, or the effort cannot be verified, report the mismatch or uncertainty instead of claiming that this request used Lite.
- Confirm Copilot code review is enabled for the account or organization and that the token can write pull-request reviews. Copilot normally submits a `COMMENT` review, not an approval.
- If the API returns `403` or `422`, stop the review path and report the permission, policy, or collaborator failure. Do not substitute an undocumented reviewer identity.

### 6. Babysit the PR

Loop until both CI and review are clean. A requested reviewer that has not submitted a review is still pending. Use bounded polls while CI runs; use the agent/runtime's wait facility between polls instead of a busy loop or an unbounded shell wait. Stop after a reasonable timeout, such as 15 minutes, and report the review as pending rather than claiming success.

**CI checks:**
```bash
gh pr checks "$PR_NUMBER" --repo "$OWNER/$REPO" > /tmp/checks.txt 2>&1 < /dev/null
cat /tmp/checks.txt
```
For any failing check, read the log (`gh run view "$RUN_ID" --repo "$OWNER/$REPO" --log-failed > /tmp/log.txt 2>&1 < /dev/null`), fix the real cause, and push a follow-up commit. Never re-run a job hoping it goes green without understanding why it was red.

**Copilot review state and comments:**
```bash
# The reviewer remains here until it submits a review.
gh api "repos/$OWNER/$REPO/pulls/$PR_NUMBER/requested_reviewers" \
  --jq '{users: [.users[].login], teams: [.teams[].slug]}' \
  > /tmp/copilot-requested.txt 2>&1 < /dev/null

# A submitted Copilot review has submitted_at and is tied to a commit_id.
gh api --paginate "repos/$OWNER/$REPO/pulls/$PR_NUMBER/reviews" \
  --jq '.[] | select(((.user.login // "") | ascii_downcase | contains("copilot-pull-request-reviewer"))) | {id, state, submitted_at, commit_id, body}' \
  > /tmp/copilot-reviews.txt 2>&1 < /dev/null

# Inline review comments.
gh api --paginate "repos/$OWNER/$REPO/pulls/$PR_NUMBER/comments" \
  --jq '.[] | select(((.user.login // "") | ascii_downcase | contains("copilot-pull-request-reviewer"))) | {id, path, line, side, body, html_url}' \
  > /tmp/copilot-inline-comments.txt 2>&1 < /dev/null

# The overview and effort-level comment is an issue comment, not an inline review comment.
gh api --paginate "repos/$OWNER/$REPO/issues/$PR_NUMBER/comments" \
  --jq '.[] | select(((.user.login // "") | ascii_downcase | contains("copilot-pull-request-reviewer"))) | {id, body, created_at, html_url}' \
  > /tmp/copilot-overview.txt 2>&1 < /dev/null
```

Poll the requested-reviewer endpoint and the reviews endpoint together. Do not treat the reviewer disappearing from `requested_reviewers` as sufficient; require a Copilot review with `submitted_at`, and require its `commit_id` to match the current PR head:
```bash
gh pr view "$PR_NUMBER" --repo "$OWNER/$REPO" --json headRefOid --jq .headRefOid
```

Triage every Copilot comment individually — **do not blanket-apply suggestions**:
1. Read the flagged code in full context.
2. Decide: real issue, or false positive?
   - **Real** → fix it in a follow-up commit on the same branch (`fix:` / `refactor:` / `chore:` as appropriate), push, and reply to the comment saying what changed.
   - **False positive** → reply on the thread explaining concretely why it doesn't apply, then resolve it. Do not change working code to silence a reviewer.
   - **Out of scope but valid** → say so on the thread and note it in the final report rather than expanding the PR.
3. Resolve the thread once handled.

Reply to an inline comment with its comment id, then resolve the review thread by its GraphQL thread id:
```bash
gh api --method POST "repos/$OWNER/$REPO/pulls/$PR_NUMBER/comments" \
  -f body='Explain the fix or why the comment does not apply.' \
  -F in_reply_to="$COMMENT_ID"

gh api graphql \
  -f query='mutation($threadId:ID!) { resolveReviewThread(input:{threadId:$threadId}) { thread { isResolved } } }' \
  -F threadId="$THREAD_ID"
```

After each follow-up push, request Copilot again with the same API call, then repeat the bounded state, review, overview, and inline-comment polls. A review for an older commit does not cover the new head.

Completion criterion: required CI checks pass; a Copilot review is submitted for the current head; the overview confirms `Lite` or the final report explicitly says that effort was not verified; and every Copilot comment is fixed, dismissed as a false positive, or explicitly deferred with a reason.

### 7. Report

Give the user: PR link and number, branch name, commit subjects, validation results (including anything that couldn't run), final CI status from actual `gh pr checks` output, and how each Copilot comment was resolved (fixed / dismissed as false positive / deferred with reason).

## Sandbox gotchas

`gh` subcommands can appear to hang or open an alternate buffer in this terminal. Always redirect and read back:
```bash
gh pr checks "$PR_NUMBER" --repo "$OWNER/$REPO" > /tmp/out.txt 2>&1 < /dev/null
cat /tmp/out.txt
```

## Guardrails

- Don't commit on top of a failing build, test, or lint run.
- Don't open the PR as a draft.
- Keep model information out of the PR title, branch, and git commit messages. The only allowed location is the single final Model information line in the PR body described in Step 4.
- Don't add validation or testing-steps sections to the PR description.
- Don't claim CI passed without reading real `gh pr checks` output.
- Don't auto-accept Copilot suggestions without validating them.
- Don't force-push, rewrite history, or amend pushed commits without explicit user confirmation.
- Don't expand scope: follow-up commits fix review findings on this change, not adjacent code.
