---
name: claude-github-code-review
description: Review a GitHub PR and post inline comments on the lines you suggest changes to. Invoke as /claude-github-code-review <PR URL or number>.
---

# /claude-github-code-review — Review a GitHub PR and post inline comments

One-shot: given a PR URL or number, fetch the diff, review it against repo conventions, and **post inline comments directly via `gh api`** as a single review. No draft step, no confirmation — the user invoked the skill expecting the review to land on GitHub.

## Usage

```
/claude-github-code-review <PR URL>
/claude-github-code-review <PR number>      # uses the current repo
```

## Workflow

### 1. Fetch PR data

```bash
gh pr view <pr> --json number,title,body,headRefName,baseRefName,author,state,files,additions,deletions
gh pr diff <pr>
```

If the PR's head branch is checked out locally, prefer `Read` on the actual files for full context. Otherwise read changed files via `gh api repos/{owner}/{repo}/contents/{path}?ref={head-sha}`.

Capture `owner`, `repo`, and `number` from the URL — needed for the review POST.

### 2. Load conventions

Before reviewing, read any `CLAUDE.md` in the repo root **and** in any subdirectory touched by the PR (e.g. `backend/CLAUDE.md`, `frontend/CLAUDE.md`). These often encode testing/style rules that are the most actionable review targets.

### 3. Analyze the diff

Look for, in priority order:

- **Critical** — bugs, security issues, broken contracts, language conventions, missing error handling at boundaries.
- **Suggestion** — convention violations from `CLAUDE.md`, missing test assertions that would let regressions slip through, naming/structure clarity wins.
- **Nitpick** — minor style; mention sparingly and only when worth the noise.

**Inspect for — these drive most of the high-value findings:**

- **Intent match** — does the diff do what the PR title/body claims? Flag scope creep (unrelated changes bundled in) and under-delivery (claimed behavior the diff doesn't actually implement).
- **Blast radius beyond the diff** — when a signature, exported type, shared helper, or config key changes, check callers/consumers *not in the diff*. Analyze past the hunks even though the resulting comment must still attach to a diff line (or the summary `body`).
- **Security** — injection (SQL/command/template), authz checks on new or changed endpoints, secrets/keys committed, unsafe deserialization, SSRF/path traversal on user-controlled input, missing input validation at trust boundaries. (Basically, OWASP Top Ten categories.)
- **Performance regressions visible in the diff** — N+1 queries or network calls in a loop, an algorithmic step gone super-linear on request-scoped/growing data, or expensive work (regex compile, IO, connection setup) repeated inside a hot loop. Only when it's structurally evident from the change — do **not** speculate about allocations, GC, or micro-optimizations that need profiling to confirm; that's noise without benchmark data.

**Skip the noise** — don't comment on generated files, lockfiles, vendored dependencies, or the mechanical fallout of a rename/move. Review the *source* of a change, not its churn; comments on mechanical churn bury the actionable findings.

For each finding, identify the **exact file path and line range in the PR diff** — comments only post on lines present in the diff hunks.

### 4. Post the review

Build a JSON payload and POST in a single review (one API call, not N separate comments). Do not pause to ask which findings to include — post everything you'd recommend.

```bash
cat > /tmp/pr_review.json <<'EOF'
{
  "event": "COMMENT",
  "body": "<short summary>",
  "comments": [
    {
      "path": "path/to/file.go",
      "line": 42,
      "side": "RIGHT",
      "body": "single-line comment"
    },
    {
      "path": "path/to/other.go",
      "start_line": 76,
      "line": 83,
      "side": "RIGHT",
      "body": "multi-line comment"
    }
  ]
}
EOF
gh api --method POST "repos/<owner>/<repo>/pulls/<number>/reviews" --input /tmp/pr_review.json -q '{id, state, html_url}'
```

Return the `html_url` to the user so they can jump straight to the review. That URL — plus a one-line summary of how many comments were posted — is the entire user-facing reply. No findings list, no analysis dump in chat.

## Rules and gotchas

- **`event` must be `COMMENT`** for feedback. Never use `APPROVE` or `REQUEST_CHANGES` — that's a human decision.
- **Single-line comment**: only `line` (the line in the new file).
- **Multi-line comment**: both `start_line` and `line`; `start_line` < `line`.
- **`side: "RIGHT"`** for added/modified lines (the new file). Use `"LEFT"` only when commenting on a deleted line.
- **Comments must hit lines in the diff hunks.** If you want to comment on a nearby unchanged line, either widen to a multi-line range that includes a changed line, or fall back to a top-level review `body` note.
- **Pass the payload via `--input <file>`**, not `-f` or inline `-r`. Escaping JSON with code blocks and newlines through a shell flag is painful and error-prone.
- **One review per invocation**, not one comment per API call — keeps the PR timeline clean.
- **Don't post anything that isn't actionable.** Praise belongs in the top-level `body` if anywhere; the inline comments should each suggest a concrete change.
- **Don't fabricate line numbers** — pull them from the diff hunks you actually fetched. If unsure where a line lives in the new file, re-read the diff before posting.
- If `gh` returns `422 Unprocessable`, the most common causes are: line not in diff, `start_line >= line`, or wrong `side`. Inspect the response body — it names the offending comment.

## Committable suggestion blocks

GitHub renders a special fenced block as a one-click "Commit suggestion" button:

````markdown
```suggestion
the exact replacement lines, indentation included
```
````

When the comment targets a single `line`, the suggestion replaces that one line. When it targets a `start_line`+`line` range, it replaces the whole range. **Use this whenever the fix qualifies** — it turns review feedback into a one-click change.

### When to include a `suggestion` block

A suggestion block is appropriate **only when all of the following hold**:

- The change is a **contiguous** edit, fully contained in the comment's line range.
- Indentation/whitespace can be reproduced exactly (tabs vs spaces, leading indentation level).
- Committing the suggestion alone leaves the file in a **buildable / passing state** — no cascading edits required elsewhere.
- The replacement is mechanical enough that the author shouldn't need to massage it.

If any of those fail, **skip the suggestion block** and use prose + a fenced ```go (or whatever language) example instead. Do not post a "suggestion" that would break the build on click — that's worse than no suggestion.

Common cases to skip:
- File renames or moves (no API for it).
- Refactors that touch multiple non-adjacent regions (e.g. add an import at the top *and* edit a function below).
- Struct/field renames where call sites elsewhere would also need updating.
- Anything where you'd need to add a new file.

### Indentation

The suggestion's contents are inserted verbatim. If the targeted lines start with two tabs, your suggestion lines must too. When in doubt, re-read the file with `Read` and match its indentation byte-for-byte.

### Building suggestion-bearing comments

Tabs and multi-line bodies make inline JSON construction error-prone. Two safe patterns:

```bash
# Pattern A: write each comment body to a tempfile, then assemble with jq
printf '%s' "$(cat <<'EOF'
Per `backend/CLAUDE.md` …

```suggestion
		newLine1
		newLine2
```
EOF
)" > /tmp/c1.md

jq -n --rawfile c1 /tmp/c1.md '{
  event: "COMMENT",
  body: "Drive-by review.",
  comments: [
    {path: "path/to/file.go", start_line: 76, line: 83, side: "RIGHT", body: $c1}
  ]
}' > /tmp/pr_review.json

gh api --method POST "repos/<owner>/<repo>/pulls/<number>/reviews" --input /tmp/pr_review.json
```

Always validate the resulting JSON (`jq . /tmp/pr_review.json`) before POSTing.

## Tone for review bodies

- Lead with the suggested change, not the problem statement.
- Cite the rule (e.g. "per `backend/CLAUDE.md`: …") when the suggestion comes from a documented convention — it makes the suggestion harder to wave off and easier to accept.
- Prefer a `suggestion` block over a prose snippet whenever the change qualifies (see above).
- Keep each comment focused on one issue. If two issues touch the same lines, post two comments.
