# pi-code-review

`/code-review` for pi: independent auditors inspect one immutable pull-request snapshot, each
finding is scored on its own, and only findings above the confidence threshold are reported.

The design follows the code-review plugin that ships with Claude Code — parallel auditors, an
independent scorer per finding, a confidence threshold, and links pinned to a full SHA. That plugin
is Anthropic's and not open source, so nothing here is copied from it: the workflow is reimplemented
for pi on top of `pi-subagents`.

## Requires

- [`pi-subagents`](https://github.com/emizuki/pi-subagents) — the `subagent` tool does the fan-out
- `gh`, authenticated, for reading the PR and optionally commenting
- `git`, used to build the immutable snapshot and inspect history

## Install

Install both packages:

```bash
pi install npm:@emizuki/pi-subagents
pi install npm:@emizuki/pi-code-review
```

That is the complete setup. `pi-subagents` discovers the review agents declared by this package,
and Pi loads the packaged `/code-review` prompt; no copy or symlink step is needed.

Install `pi-code-review` at user scope, as shown above. Package agents follow their package's own
install scope, and this workflow dispatches every auditor at the default `agentScope: "user"`, so a
project-scope install (`pi install --local`) would leave its five agents undiscoverable.

To track both development branches directly instead:

```bash
pi install git:github.com/emizuki/pi-subagents
pi install git:github.com/emizuki/pi-code-review
```

## Safe use

A review reads untrusted PR content. Use this launcher so the checkout cannot supply parent context
or executable project resources, and so Pi plus all retained subagent transcripts inherit an
owner-only temporary directory and umask:

```bash
set -euo pipefail
cd /path/to/repository
review_tmp="$(mktemp -d "${TMPDIR:-/tmp}/pi-code-review-parent.XXXXXX")"
review_tmp="$(cd "$review_tmp" && pwd -P)"
chmod 700 "$review_tmp"
printf 'Private review tmp: %s\n' "$review_tmp"
cleanup_review_tmp() { rm -rf -- "$review_tmp"; }
trap cleanup_review_tmp EXIT HUP INT TERM
(
  umask 077
  PI_CODE_REVIEW_SAFE=1 TMPDIR="$review_tmp" \
    pi --no-context-files --no-approve
)
```

This launcher is the actual security boundary. `/code-review` performs defense-in-depth environment
checks to catch accidental direct invocation, but Markdown running inside an already-compromised
process cannot certify how Pi was started. Direct invocation outside this launcher is unsupported.
A hard `SIGKILL` cannot execute the shell trap, but any residue remains beneath the printed
owner-only directory.

Then invoke the command:

```text
/code-review              # print the review here
/code-review --comment    # post a completed review as one PR comment
```

Pi is not a sandbox. For untrusted repositories or unattended review, run it in a container or VM
with only the required repository and credentials mounted.

## What it does

1. **Pins the review identity.** It records the exact base/head OIDs, authenticated reviewer, and a
   completion marker scoped to that base/head pair.
2. **Builds immutable ground truth.** A private temporary clone is checked out at the recorded head;
   both the diff and changed-file list use the PR's merge-base comparison.
3. **Collects active rules.** It mirrors Pi's per-directory precedence:
   `AGENTS.override.md`, `AGENTS.md`, `AGENTS.MD`, `CLAUDE.md`, then `CLAUDE.MD`.
4. **Audits in parallel.** Four user-scoped agents run with fresh contexts and a safe empty `cwd`:
   one guideline auditor, two differently framed defect hunters, and one history analyst.
5. **Requires complete coverage and output.** Every auditor must read the complete diff through EOF
   in chunks. Missing, failed, malformed, incomplete, or truncated dimensions fail closed.
6. **Scores each finding alone.** Scorers run in fresh contexts, at most eight per batch, and every
   label must receive exactly one valid score.
7. **Reports scores of 80 or above.** A fully completed clean review stays silent. An incomplete
   review reports its failure and never posts a completion marker.
8. **Posts per revision.** `--comment` uses a same-author marker containing full base and head OIDs,
   and rechecks the complete identity before and after posting.
9. **Cleans up privately.** The diff, clone, comment body, and retained child sessions live beneath
   the launcher's owner-only `TMPDIR` and are removed on controlled exit.

## Model policy

Discovery quality controls recall: a strong scorer can reject a false positive but cannot recover a
bug an auditor never found. All four auditors and every confidence scorer therefore omit `model`
and inherit the already-working session model, matching `pi-subagents` guidance for judgement work.

Use a cheaper model only for separate mechanical reconnaissance such as listing or locating files.
To lower review cost, deliberately choose a cheaper session model and accept the corresponding
recall tradeoff rather than silently weakening selected review dimensions.

`pi-subagents` still supports explicit model selection: a top-level `model` applies to all tasks and
a per-task `model` overrides it. Any override must use an offered model and handle provider/account
rejection at runtime; omitting it remains this workflow's safe default.

## Why it is shaped this way

**Two defect hunters, one guideline auditor.** Guideline findings already receive a second opinion
from the scorer, which reopens the file and verifies the quoted rule. Defect findings have no such
backstop, and local defects differ from cross-file interaction defects, so those get separate
framings.

**Scoring asks “is this true?”, not “does this matter?”.** A trivial defect that certainly exists
scores above an important one that is merely plausible. The scorer verifies the source, exact diff,
rule text, and history before assigning confidence.

**One scorer per finding, blind to the rest.** Every scorer receives `context: "fresh"`, one labelled
finding, and no sibling scores. This avoids list-relative calibration.

**The snapshot is separate from child startup.** Children start in an empty private directory and
receive the repository snapshot as an absolute data path. Their Pi processes therefore do not load
PR-controlled `.pi` resources, while Git inspection still targets the exact detached snapshot.

**Silence is reserved for a verified clean run.** Operational failure, malformed output, truncation,
or an unscored finding is reported explicitly and cannot create a completion marker.

**Comments are comparison-scoped.** The marker contains full base and head OIDs and is accepted only
from the authenticated reviewer. A base retarget, new head, or marker from another author cannot
suppress a new review.

## Adapting it

The confidence threshold and model policy live in `prompts/code-review.md`. `pi-subagents` permits
at most eight tasks per parallel call and runs four concurrently, so the four auditors fit one wave
and scorers are batched in eights. The workflow caps scoring at 24 findings; a larger result is
reported as partial and is never marked complete.
