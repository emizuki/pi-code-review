---
name: code-review
description: Review a pull request with parallel auditors and confidence scoring, then report only what survives
---

Review the current pull request. Argument: `$ARGUMENTS` — if it contains `--comment`, post the
result to the PR; otherwise print it here and post nothing.

Run the whole orchestration yourself. A subagent cannot dispatch subagents, so every `subagent`
call below is made from this session.

## 1. Decide whether to review at all

Read the PR with `gh pr view --json number,title,state,isDraft,headRefOid,baseRefName,files,comments`.

Stop, and say which of these applied, when the PR is closed or merged, is a draft, changes nothing
but generated files or lockfiles, or already carries a review comment from this command. A second
opinion nobody asked for is worse than none.

## 2. Collect the ground truth

- The diff: `gh pr diff` for the patch, and `git diff --name-only $(gh pr view --json baseRefName -q .baseRefName)...HEAD` for the file list.
- The house rules: every `AGENTS.md` and `CLAUDE.md` from the repository root down to the
  directories the diff touches. A rule in a subdirectory beats one above it.
- The head SHA, so links can point at an immutable blob rather than a moving branch.

Write a short factual summary of what the change does. The auditors get it, and a summary that
editorialises leads four agents astray at once.

## 3. Audit in parallel

One `subagent` call, four tasks, so they run concurrently and independently. Give each the diff,
the summary, and the guideline files — not each other's opinions.

```
{ "tasks": [
  { "agent": "guideline-auditor", "task": "<diff + guidelines>: find violations" },
  { "agent": "guideline-auditor", "task": "<the same, independently>" },
  { "agent": "bug-finder",        "task": "<diff + summary>: find defects introduced here" },
  { "agent": "history-analyst",   "task": "<changed files>: find what the history warns about" }
] }
```

Two guideline auditors is not a typo. They run on the same input without seeing each other, so
agreement is evidence and disagreement is a flag — the cheapest redundancy available.

Pass `model` on the call to run these on a cheaper model than this session; they read and report,
which is the work cheap models do well.

## 4. Score every finding, independently

Deduplicate first: several auditors reporting one fault is one finding, and that agreement is
worth recording.

Then one `confidence-scorer` per finding, in parallel, each seeing **one** finding and the code —
never the other findings, never the score of another. Batch in groups of 8, since parallel mode
takes at most 8 tasks.

A scorer returns a number from 0 to 100 and one sentence of justification.

## 5. Report what survives

Keep findings scoring **80 or above**. Below that is noise wearing the costume of rigour.

Say nothing if nothing survives — no "looks good", no summary of what you checked. Silence is the
correct output for a clean change, and a reviewer that always finds something teaches people to
ignore it.

Otherwise, ranked by score:

```markdown
**`path/to/file.ts:42`** — one-line claim (confidence: 88)

What breaks, and the input or sequence that breaks it.

https://github.com/<owner>/<repo>/blob/<full-sha>/path/to/file.ts#L42-L48
```

Full SHA in links, never a branch name: a branch link rots the moment someone pushes.

With `--comment`, post it as one comment with `gh pr comment --body-file`, not as a review, and
not as one comment per finding. Without it, print and stop — do not post, and do not offer to.

## 6. Offer the fix, do not make it

List what you would change. Apply nothing unless asked: a review that edits the code it is
reviewing has stopped being a review.

If the change came from a `subagent` run whose id is still resumable, say so — resuming the agent
that wrote the code is cheaper and better informed than a fresh one reading the diff cold.
