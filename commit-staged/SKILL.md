---
name: commit-staged
description: Use this skill when the user asks to commit only their currently staged git changes — for example "commit the staged changes", "commit what I have staged", or "help me commit (ignore chat history)". This skill ignores prior conversation context and derives the commit message strictly from `git diff --staged`. Do NOT use it when the user wants to stage files first, amend, push, or commit everything including unstaged/untracked files.
---

# commit-staged

Commit the user's **currently staged** git changes on the **current branch**, without creating or switching branches. **Ignore prior chat history** — do not assume context from earlier in the conversation about what was changed or why. Derive everything from the staged diff itself.

## Steps

1. Run these in parallel:
   - `git status` (never use `-uall`) — see what is staged vs. unstaged vs. untracked.
   - `git diff --staged` — see the exact content being committed.
   - `git log -n 10 --oneline` — match this repo's commit message style.

2. If **nothing is staged**, stop and tell the user. Do not stage anything yourself, and do not fall back to `git add -A` or `git add .`.

3. Analyze **only the staged diff**:
   - Classify the change (feat / fix / refactor / docs / test / chore / etc.) from what the diff actually shows.
   - Draft a concise commit message (1–2 sentences) focused on the *why*, in the style observed from recent commits.
   - Do **not** mention unstaged or untracked files in the message.
   - Do **not** pull motivation from chat history — if the *why* isn't evident from the diff, say so and ask the user.

4. Show the user:
   - The list of staged files (from `git status`).
   - The drafted commit message, including any co-author trailer. Format the trailer as `Co-Authored-By: <agent name and known model> <attribution email>` with the email enclosed in angle brackets. Use the actual agent name and model when reliably available from the active runtime, and an attribution email supplied by the runtime or explicit configuration. Do not infer identity from this skill, previous commits, or staged content. Omit unknown model details; if a reliable attribution name and email are unavailable, omit the trailer rather than inventing them.
   - Then ask: **"Commit with this message? (y / edit / n)"**

5. Only after the user confirms, run `git commit` on the current branch using a HEREDOC for the message:

   ```bash
   git commit -m "$(cat <<'EOF'
   <drafted message>

   <optional Co-Authored-By trailer resolved above; omit this line and its preceding blank line if unavailable>
   EOF
   )"
   ```

6. Run `git status` once more to confirm. Report the new commit hash.

## Hard rules

- **Never** create or switch branches. If HEAD is detached, stop and ask the user to check out the intended existing branch before committing.
- **Never** run `git add` — only commit what is already staged.
- **Never** use `--no-verify`, `--amend`, or `-i`.
- **Never** push.
- If a pre-commit hook fails, fix the underlying issue and create a **new** commit (do not amend).
- If files that look like secrets (`.env`, `*credentials*`, `*.pem`, `*.key`) are staged, warn the user and wait for explicit confirmation before committing.
