---
name: commit-message-writer
description: Generates structured commit messages following the Conventional Commits
  standard. Use when you want to commit your changes and need a well-formatted message.
  Triggers on "write a commit message", "commit my changes", "summarize my staged
  diff", "what should my commit say", or any request to describe or document staged
  changes for version control.
---

# commit-message-writer

You generate structured commit messages from staged git changes.

## How to invoke

Run `git diff --staged` to read the staged changes. If nothing is staged, tell the
user and suggest they run `git add` first.

Generate first. Do not ask clarifying questions before producing the commit message.
If you need to make assumptions about scope or type, make them and note them after
the output.

## Output format

~~~
type(scope): short description
 
[body — optional, include if changes are non-trivial]
 
[footer — optional]
~~~

**Type** — choose one:
- `feat` — a new feature
- `fix` — a bug fix
- `docs` — documentation changes only
- `refactor` — code change that neither fixes a bug nor adds a feature
- `test` — adding or updating tests
- `chore` — build process, tooling, or dependency updates

**Scope** — the module, file, or area affected. Use the directory name or component
name. Omit if the change spans the entire codebase.

**Short description** — imperative mood, under 72 characters, no period at the end.
"Add user authentication" not "Added user authentication" or "Adds user authentication."

**Body** — what changed and why, not how. One bullet per logical change. Skip if the
short description is self-explanatory.

**Footer** — include `BREAKING CHANGE:` if the commit breaks backward compatibility.
Include `Closes #N` if it resolves a GitHub issue.

## Quality rules

- Never use "updated", "changed", or "modified" in the short description — be specific
- Never write "various improvements" or "misc fixes" — name what improved
- If more than three files changed across unrelated concerns, flag it:
  "These changes may be better split into separate commits: [list concerns]"
- The short description must be under 72 characters — count before outputting

## Example output

Input: staged changes adding a rate limiter to an API endpoint

~~~
feat(api): add rate limiting to /query endpoint
 
- Limits requests to 100 per minute per IP using Cloudflare's rate limit binding
- Returns 429 with Retry-After header when limit is exceeded
- Adds rate limit configuration to wrangler.toml
 
Closes #47
~~~