---
name: code-review
description: Review a pull request with parallel auditors and independent confidence scoring, then report only what survives
argument-hint: "[--comment]"
---

Review the current pull request. Arguments: `$ARGUMENTS` — post to the PR when they contain
`--comment`, otherwise print here and post nothing.

Run the orchestration yourself. A subagent cannot dispatch subagents, so every `subagent` call
below is made from this session.

## 1. Decide whether to review at all

```bash
gh pr view --json number,title,state,isDraft,headRefOid,baseRefName,url,comments
```

Stop, saying which applied, when the PR is closed or merged, is a draft, changes nothing but
generated files or lockfiles, or already carries a comment whose body contains the marker
`<!-- pi-code-review -->`. That marker is how a review recognises its own work; a comment without
it belongs to a person.

## 2. Collect the ground truth

```bash
gh pr diff > /tmp/pr-<number>.diff          # the patch, by PR ref, not a local branch
gh pr diff --name-only                      # exact changed files, needs no local ref
```

Use `gh pr diff`, never `git diff base...HEAD`: `baseRefName` is a bare local branch that is
usually stale after checkout and sometimes absent, and a stale base silently widens the file list
to things this PR never touched.

Note the head SHA from step 1 — links need it — and find every `AGENTS.md` and `CLAUDE.md` from
the repository root down to the touched directories. A rule in a subdirectory beats one above it.

Write a short factual summary of what the change does. It goes to every auditor, so a summary that
editorialises leads all of them astray at once.

## 3. Audit in parallel

Pass the diff by **path**, never by pasting it. A task string becomes a single argv entry, and a
patch plus a few rule files will exceed the limit and fail the spawn. Every agent can `read`.

```json
{
  "model": "<cheapest model the model parameter offers>",
  "tasks": [
    { "agent": "guideline-auditor",  "task": "Diff: /tmp/pr-123.diff. Rules: <paths>. Summary: <summary>. Report violations of those rules." },
    { "agent": "bug-finder",         "task": "Diff: /tmp/pr-123.diff. Summary: <summary>. Report defects introduced by these lines." },
    { "agent": "interaction-finder", "task": "Diff: /tmp/pr-123.diff. Summary: <summary>. Report what elsewhere in the repository this change breaks." },
    { "agent": "history-analyst",    "task": "Diff: /tmp/pr-123.diff. Base: <baseRefName>. Head: <headRefOid>. Report what the history of these lines warns about." }
  ]
}
```

Two defect hunters, one guideline auditor. Guideline findings already get a second opinion from
the scorer, which re-opens the file and checks the quoted rule; defect findings get no such check,
and two hunters with **different framings** find different things — one reads what the change does
wrong, the other reads what it breaks somewhere else. Two copies of one framing would mostly agree
with themselves.

The auditors read and report, which cheap models do well. The scorers in step 4 do not take
`model` — that step is judgement, and it decides what you see.

## 4. Turn output into findings

Parse each result into blocks. Discard anything that is not a well-formed block: an auditor that
answers in prose has given you an impression, and inventing a finding from it reintroduces upstream
the exact failure its own prompt forbids. Say how many you discarded.

If a task's header says `failed`, say so and name the dimension that went unreviewed. Do not
silently proceed as if four ran.

Merge duplicates — one fault reported by two agents is one finding — and note which agents agreed.

**Number the surviving findings `F1`, `F2`, … and keep that list.** Parallel results come back
without the task text, so `F<n>` is the only thing tying a score to the finding it scored.

If there are none, go to step 5 now. Do not call the scorer: a call with an empty `tasks` array is
rejected as no mode selected, and a clean PR is the case this is proudest of.

## 5. Score every finding, independently

One `confidence-scorer` per finding, batched in groups of 8 — the parallel cap — with the batch
kept in `F<n>` order:

```json
{ "tasks": [
  { "agent": "confidence-scorer", "task": "FINDING F1: <the finding>. Diff: /tmp/pr-123.diff. Score it." },
  { "agent": "confidence-scorer", "task": "FINDING F2: <the finding>. Diff: /tmp/pr-123.diff. Score it." }
] }
```

Each scorer sees one finding, never another's finding and never another's score. It echoes the
`F<n>` back, so match scores by that label and not by position. A score you cannot match to a
finding is a score you discard.

Stop at 24 findings. More than that is not a review, it is a rewrite, and scoring them all costs
more than reading the diff yourself — say how many went unscored.

## 6. Report what survives

Keep findings scoring **80 or above**. Below that is noise wearing the costume of rigour.

Say nothing when nothing survives. No "looks good", no list of what you checked. Silence is the
right output for a clean change, and a reviewer that always finds something teaches people to stop
reading it.

Otherwise, ranked by score, one block each:

```markdown
**`path/to/file.ts:42`** — one-line claim (confidence: 88, found by 2 agents)

For a defect: what breaks, and the input or sequence that breaks it.
For a rule violation: the rule, quoted, and what the change does against it.
For a history finding: the commit, and how this change runs against it.

https://github.com/<owner>/<repo>/blob/<head-sha>/path/to/file.ts#L42
```

Do not invent a line range. Link the single line an agent reported; widen to `#L42-L48` only when
a finding genuinely spans lines you can name. A finding about a whole file takes the path with no
line, and the permalink without an anchor.

Use the full head SHA, never a branch: a branch link points at different code as soon as anyone
pushes. Drop the "found by N agents" clause when only one found it.

With `--comment`, post once with `gh pr comment --body-file`, as a comment and not a review, and
begin the body with `<!-- pi-code-review -->` so step 1 can recognise it next time. Without the
flag, print and stop — do not post, and do not offer to.

## 7. Offer the fix, do not make it

List what you would change. Apply nothing unless asked: a review that edits the code it reviewed
has stopped being a review.

If the change came from a `subagent` run in this session, check `{ "action": "runs" }` first and
only offer to resume an id that reports `resumable`. Retention is per session and is cleared by
`/new`, `/resume` and `/fork`, so an id remembered from the transcript is usually gone.
