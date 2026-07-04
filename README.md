# agent-skills

A repository of custom skills for Claude Code. It currently contains four code-review-related skills.

## Background

The four skills are based on the PR review skills (`/review-pr` and `/improve-review-pr`) introduced in the Mercari engineering blog post "[修正PRを食べてレビュースキルが賢くなる：Claude Codeによる自己改善サイクル](https://engineering.mercari.com/blog/entry/20260616-0f89ad4c7b/)" (How a review skill gets smarter by eating fix PRs: a self-improvement cycle with Claude Code). On top of the original "review a PR" / "learn missed patterns from fix PRs" self-improvement cycle, they add two capabilities:

- **Reviewing uncommitted changes** (`review-uncommitted`) — review the working tree (staged / unstaged / untracked) with the same set of perspectives as a PR review, before a PR even exists
- **Adding review perspectives from a session** (`improve-review-from-session`) — extract review perspectives not only from fix-PR analysis but also from the current conversation session: bugs that surfaced, user corrections, and non-obvious constraints

## Skills

| Skill | Role |
|---|---|
| [review-uncommitted](skills/review-uncommitted/SKILL.md) | Review uncommitted changes (perspective consumer) |
| [review-pr](skills/review-pr/SKILL.md) | Review a pull request (perspective consumer) |
| [improve-review-from-session](skills/improve-review-from-session/SKILL.md) | Extract review perspectives from the session (perspective producer) |
| [improve-review-from-pr-gap](skills/improve-review-from-pr-gap/SKILL.md) | Extract review perspectives from a fix-PR gap (perspective producer) |

### review-uncommitted

Collects every uncommitted change in the working tree (unstaged, staged, and new untracked files) and reviews it with parallel subagents against eight generic perspectives (code-review, ai-antipattern, security, edge cases and failure paths, backward compatibility, functional impact, test sufficiency, unit test coverage) plus the project-specific perspectives in `./.review/*.md`. Overlapping findings are deduplicated and merged into a single report grouped by severity (Critical / Major / Minor). The skill is read-only: it never edits code or performs git operations.

### review-pr

Fetches a PR's metadata and diff with `gh pr view` / `gh pr diff`, then applies the same perspective set as review-uncommitted (the eight generic perspectives plus `./.review/*.md`) to the PR diff using parallel subagents. The difference from review-uncommitted is the review target: the PR diff rather than the local working tree. It only reports findings — it never edits code or posts comments on the PR.

### improve-review-from-session

Looks back over the current session for material such as bugs that surfaced after the implementation seemed done, corrections from the user, and non-obvious project constraints, generalizes them into reusable checks that would apply to future, unrelated diffs, and saves each one as `./.review/<name>.md`. If an existing perspective already covers the lesson, it strengthens that file instead of creating a near-duplicate. Because the conversation history itself is the source material, the main agent performs this inline rather than delegating to a subagent.

### improve-review-from-pr-gap

For the case where an oversight was found after merge and a fix PR was needed, this skill compares the original PR with its fix PR to analyze what was missed. It only targets gaps that were actually detectable at review time by reading the original diff, and writes the reusable check that would have caught them to `./.review/*.md` (or strengthens an existing perspective that should have caught them). Anything that could only be discovered from runtime behavior is not turned into a perspective, and the report says so explicitly.

## How it works: accumulating perspectives in `./.review/`

The four skills cooperate through `./.review/*.md` (one perspective = one file) placed at the root of the project under review.

- **Producers**: `improve-review-from-session` and `improve-review-from-pr-gap` write lessons extracted from sessions and fix PRs as perspective files
- **Consumers**: `review-uncommitted` and `review-pr` automatically load the relevant perspective files at review time and apply them on top of the generic perspectives

Perspective files share a common format (frontmatter: `name` / `description` / `origin`; body: When to apply / What to check / Background / Examples). At review time only the frontmatter and "When to apply" section are pre-read to filter perspectives by relevance to the diff, so unrelated perspectives never add noise. This loop — lessons from one review becoming perspectives for the next — corresponds to the self-improvement cycle described in the original blog post.
