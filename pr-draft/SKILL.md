---
name: pr-draft
description: Use this skill when the user asks to draft, generate, or write a pull request title and description for the current branch — for example "draft a PR", "generate PR description", "write up a PR for this branch". This skill diffs the current branch against the repository's default branch (origin/HEAD) and produces a PR title and description from that diff. Do NOT use it when the user wants to actually create/open the PR on GitHub/Azure DevOps (use `gh pr create` flow instead), commit changes, or push.
---

# pr-draft

Draft a pull request **title and description** for the current branch by diffing it against the repository's default branch. This skill only generates the text — it does **not** create the PR, push, or commit.

## Steps

1. Discover the default (main) branch. Run via the **Bash tool** (the command uses POSIX `sed`):

   ```bash
   main=$(git symbolic-ref refs/remotes/origin/HEAD | sed s@^refs/remotes/origin/@@); echo "$main"
   ```

   If this fails (e.g. `origin/HEAD` is not set), tell the user and ask which branch to diff against — do not guess.

2. Gather context. Run in parallel:
   - `git rev-parse --abbrev-ref HEAD` — confirm the current branch (and that it is not the default branch itself).
   - `git log "$main"..HEAD --oneline` — every commit on this branch since it diverged.
   - `git diff "$main"...HEAD --stat` — files changed and size of changes.
   - `git diff "$main"...HEAD` — the full diff for content analysis.
   - `git log -n 10 --oneline "$main"` — recent default-branch commits to match this repo's PR/commit style.

   Note: use the **three-dot** form (`$main...HEAD`) so the diff is against the merge base, not the current tip of main.

3. If the current branch **is** the default branch, or `git log $main..HEAD` is empty, stop and tell the user — there is nothing to draft.

4. Analyze **all** commits and the full diff (not just the latest commit) to understand the full scope of the change.

5. Draft the PR:
   - **Title:** under 70 characters, imperative mood, no trailing period. Match the style of recent commits in this repo.
   - **Description:** a `## Summary` section (1–3 bullets focused on the *why*, not a file-by-file recap) and a `## Test plan` section (bulleted checklist of what to verify). Keep it tight — reviewers should be able to read it in under a minute.

6. Output the drafted title and description to the user in a code block so they can copy it. Do **not** call `gh pr create`, `az repos pr create`, or any other PR-creation command. Do **not** push.

7. If the user then asks to actually open the PR, follow the standard PR-creation flow (which is outside this skill's scope).

## Hard rules

- **Never** push, commit, amend, or stage files.
- **Never** call `gh pr create` or any PR-creation API from within this skill — output only.
- **Never** pull motivation from chat history if it is not supported by the diff or commit messages; if the *why* is unclear, say so and ask the user.
- If the diff is empty or the branch equals the default branch, stop and report — do not invent a PR.
