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

Agents are not a pi resource type, so link them yourself:

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
3. **Audits in parallel**, four tasks in one `subagent` call: two `guideline-auditor` runs, one
   `bug-finder`, one `history-analyst`. None of them sees another's output.
4. **Scores each finding alone.** One `confidence-scorer` per finding, batched in groups of eight,
   each seeing a single finding and the code — never another finding, never another score.
5. **Reports what scores 80 or above**, ranked, and says nothing at all when nothing does.

## Why it is shaped this way

**Two guideline auditors on identical input.** They cannot see each other, so agreement is
evidence and disagreement is a flag. It is the cheapest redundancy available, and guideline checks
are where confident nonsense is most common.

**Scoring asks "is this true", not "does this matter".** A trivial defect that certainly exists
outranks an important one that might not. The scorer's main job is to open the file and check that
the claimed code is there, and for a guideline finding, that the quoted rule contains the words the
finding needs it to contain — the single check that removes most bad findings.

**One scorer per finding, blind to the rest.** A scorer that sees a list calibrates against the
list. A scorer that sees one finding has to look at the code.

**Silence on a clean change.** No "looks good", no summary of what was checked. A reviewer that
always finds something teaches people to stop reading it.

**Links carry the full SHA.** A branch link points at different code the moment someone pushes.

## Adapting it

The threshold lives in step 5 of `prompts/code-review.md`; lower it to see what is being
discarded. The auditor set lives in step 3 — adding a dimension means adding a task and an agent
file. `pi-subagents` caps a parallel call at 8 tasks with 4 running at once, which is why the
scorers are batched.

Run the auditors on a cheap model by passing `model` on the `subagent` call: they read and report,
which is what cheap models are good at. Leave the scorers on the session model — that step is
judgement, and it is the step that decides what you see.
