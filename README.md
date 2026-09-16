# pi-code-review

`/code-review` for pi: several auditors read a pull request independently, every finding is scored
on its own, and only what survives the threshold is reported.

The design follows the code-review plugin that ships with Claude Code — parallel auditors,
redundant guideline checks, one independent scorer per finding, a confidence threshold, and links
pinned to a full SHA. That plugin is Anthropic's and not open source, so nothing here is copied
from it: the workflow is reimplemented for pi, in its own words, on top of `pi-subagents`.

## Requires

- [`pi-subagents`](https://github.com/emizuki/pi-subagents) — the `subagent` tool does the fan-out
- `gh`, authenticated, for reading the PR and optionally commenting
- `git`, which the history analyst and the scorer both shell out to

## Install

```bash
pi install git:github.com/emizuki/pi-code-review
```

Agents are not a pi resource type, so link those yourself, from a clone:

```bash
mkdir -p ~/.pi/agent/agents
for f in agents/*.md; do ln -sf "$PWD/$f" ~/.pi/agent/agents/; done
```

## Use

```
/code-review              # print the review here
/code-review --comment    # post it on the PR as one comment
```

## What it does

1. **Decides whether to review.** Closed, merged, draft, generated-only, or already reviewed by
   this command: it stops and says which.
2. **Collects ground truth** — the diff, every `AGENTS.md` and `CLAUDE.md` from the root down to
   the touched directories, and the head SHA for links.
3. **Audits in parallel**, four tasks in one `subagent` call: one `guideline-auditor`, two defect
   hunters with different framings (`bug-finder` on what the change does wrong, `interaction-finder`
   on what it breaks elsewhere), and one `history-analyst`. None sees another's output. The diff is
   passed by path, not pasted — a task is one argv entry and a patch overflows it.
4. **Turns output into findings**, discarding anything that is not a well-formed block, naming any
   dimension whose task failed, merging duplicates, and labelling the survivors `F1`, `F2`, …
5. **Scores each finding alone.** One `confidence-scorer` per finding, batched in eights, each
   seeing one finding and the diff — never another finding, never another score — and echoing its
   label back, because parallel results arrive without their task text.
6. **Reports what scores 80 or above**, ranked, and says nothing at all when nothing does.

## Why it is shaped this way

**Two defect hunters, one guideline auditor.** Claude Code's plugin doubles the guideline check,
because that is where a model most confidently invents a rule. Here the scorer already re-opens the
file and checks that the quoted rule contains the words the finding needs it to contain, so the
second auditor would be checking something already checked. Defect findings have no such backstop,
and defects are where the variance is: two hunters reading the same code independently find largely
different things. So the budget moved.

The two hunters are deliberately **not** copies. One reads what the change does wrong; the other
reads what it breaks somewhere else — a caller left unupdated, a branch inserted ahead of the one
that used to answer, a condition that can no longer be false. Those need two files open at once and
are invisible in a patch. Two copies of one framing would mostly agree with themselves.

**Scoring asks "is this true", not "does this matter".** A trivial defect that certainly exists
outranks an important one that might not. The scorer opens the file, checks the claimed code is
there, and for a guideline finding checks the quoted rule contains the words the finding needs it
to contain — the single check that removes most bad findings. It also has `git`, so it can ask
whether the change introduced the defect at all; a pre-existing bug attributed to a PR buries the
findings that belong to it.

**One scorer per finding, blind to the rest.** A scorer that sees a list calibrates against the
list. A scorer that sees one finding has to look at the code.

**Silence on a clean change.** No "looks good", no summary of what was checked. A reviewer that
always finds something teaches people to stop reading it.

**Links carry the full SHA.** A branch link points at different code the moment someone pushes.
Line ranges are not invented: an agent reports one line, so the link points at one line.

**Posted comments carry a marker.** `<!-- pi-code-review -->` is how the next run recognises its own
work and declines to review twice. Without it, a comment from this command is indistinguishable from
a human's.

**The agents do not inherit the repository's `AGENTS.md`.** They are handed the rules as data to
audit against; loading them again as instructions would let a pull request under review put text in
front of the scorer that decides whether its own findings get published.

## Adapting it

The threshold lives in step 6 of `prompts/code-review.md`; lower it to see what is being
discarded. The auditor set lives in step 3 — adding a dimension means adding a task and an agent
file, and four is the number that fits the concurrency limit without queueing. `pi-subagents` caps
a parallel call at 8 tasks with 4 running at once, which is why the scorers are batched and why
step 5 stops at 24 findings.

Run the auditors on a cheap model by passing `model` on the `subagent` call: they read and report,
which is what cheap models are good at. Leave the scorers on the session model — that step is
judgement, and it is the step that decides what you see.
