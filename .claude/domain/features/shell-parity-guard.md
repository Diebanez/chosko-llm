# Shell parity guard

A mechanical check that the bash and PowerShell implementations of the CLI
still expose the same surface, plus the authoring rule that makes acting on it
obligatory. It is the answer to the one objection the technical direction
raises against native Windows support, and the only feature in this set whose
purpose is to stop something rather than to enable it.

## Purpose

[`../technical-direction.md`](../technical-direction.md) records native
Windows support as an explicitly open decision and names the deciding cost
precisely: *"The cost that matters is ongoing maintenance of a second
implementation, not the initial effort."* Every other feature in this set
spends that cost. This one contains it.

Drift in a dual implementation is not dramatic. Nobody decides to let the
branches diverge; someone adds a flag to `cmd-add.sh`, ships it, and it is
correct — for the platform they were on. The divergence is discovered months
later by a Windows user hitting a command that behaves differently from the
documentation, with no way to tell whether it is a bug or an intended
difference. By then several such gaps have accumulated and closing them is a
project rather than an edit.

The repository already answers this class of problem once.
`scripts/check-task-parity.sh` verifies that the task status vocabulary stays
consistent across the files that duplicate it — a small mechanical guard,
invoked at authoring time, no CI and no dependencies. This feature applies the
same pattern to a much larger duplication.

## Scope and non-goals

**In scope.** A guard script that compares the two implementations' surfaces
and reports differences, and the authoring-guide rule stating when it runs and
what a failure obliges.

**Out of scope, deliberately:**

- **Behavioural equivalence testing.** The guard compares surfaces — which
  subcommands exist, which helpers exist, which flags are accepted. It cannot
  tell whether `cmd-add.ps1` installs correctly, and pretending otherwise
  would be worse than not checking, because a passing guard would be read as a
  guarantee it does not provide. The product has no test suite by design, and
  this feature does not smuggle one in.
- **CI.** Nothing in this repository runs on a server. The guard is invoked by
  Claude at authoring time, matching how `check-task-parity.sh` is used today.
- **Blocking commits.** No git hook, no `--no-verify` to work around. A
  failing guard is information the author acts on, in a product whose whole
  gate philosophy is that the director decides.
- **Guarding the unported surface.** `task-impl` and `lib-task-external` are
  bash-only by decision. The guard must know that and treat their absence as
  correct — a guard that cries wolf about a deliberate choice gets ignored,
  and an ignored guard is worse than none.

## Architecture

Built on the stack recorded in
[`../technical-direction.md`](../technical-direction.md): a shell script under
`scripts/`, POSIX-ish bash with awk, sed, and grep, no new dependencies. It
does not source `lib.sh`, since it must be able to run against a repository
whose library is exactly what it is inspecting.

It sits alongside `scripts/check-task-parity.sh` as the repository's second
mechanical guard, and is deliberately built to the same modest standard:
readable, fast, no state, exit non-zero with a list of differences.

### What it compares

Three surfaces, each extracted from both branches by pattern-matching the
source and compared as sets:

- **Subcommands.** The routing table in `bin/chosko-llm` against the one in
  `bin/chosko-llm.ps1`, less the known-unported list. A subcommand added to
  bash and forgotten in PowerShell is the highest-consequence drift in the
  set: it turns into "unknown subcommand" for every Windows user, which reads
  as the feature not existing.
- **Helper functions.** The functions defined in `scripts/lib.sh` against
  those in `scripts/lib.ps1`. This is the largest surface and the one most
  likely to drift quietly, because a helper added for one subcommand is easy
  to add on one branch only.
- **Accepted flags.** The flag literals each `cmd-*.sh` recognises against its
  `.ps1` counterpart. This catches the most common real-world divergence: a
  new option shipped on the platform its author was using.

The comparison is symmetric. A helper present only in PowerShell is reported
too — it means either an unported bash feature or an improvement applied to
one branch, and both are drift.

### Known-unported is data, not silence

The subcommands and helpers deliberately absent from the PowerShell branch are
declared explicitly, in one place, rather than being special-cased in the
comparison logic. Two properties follow. The list is reviewable — a reader can
see what the port does not cover without reading the guard's code. And adding
to it is a visible act: excluding a newly written subcommand from parity
becomes an edit someone makes on purpose, not a check that quietly never
fired.

### The naming rule the comparison depends on

Comparing function sets requires knowing which PowerShell function corresponds
to which bash one. That mapping must be derivable by rule — either bash names
kept verbatim, or a single mechanical transformation to `Verb-Noun` — because
a hand-maintained lookup table is itself a third thing that drifts, and it
drifts silently in the one component whose job is to detect drift.

This is the same reasoning that puts the naming decision in
[`powershell-shared-lib`](./powershell-shared-lib.md)'s open questions with
this feature as its consumer: the library can be written either way, but only
one of the two is checkable, so the guard's requirement decides it.

### The authoring rule

The guard is only half the mechanism. The other half is a rule in
[`../../docs/authoring-guide.md`](../../docs/authoring-guide.md), which is the
repository's source of truth for authoring: **a change to the CLI surface
moves both branches, and the guard is run before the change is committed.**

The guide is where a change to `lib.sh` is already prompted to consider the
frontmatter schema, so it is where a change to the CLI is prompted to consider
the second implementation. Placing the rule in a document nobody reads at the
right moment would make it decoration.

## Data and state

The guard holds no state — no baseline file, no recorded set of known
differences, no cache. It reads the two implementations and reports. This is
deliberate and matches the product's rule that the filesystem is the state: a
checked-in baseline is a file that goes stale, gets regenerated to make the
guard pass, and ends up recording the drift instead of preventing it.

The known-unported list is the sole exception, and it is configuration rather
than state — a small declared fact about the port's intended shape, versioned
with the repository and changed only by a human decision.

## Interfaces and contracts

**Exposes** a single script invoked with no arguments. Exit 0 when the
surfaces match; non-zero with a readable report of the differences otherwise —
what is missing, from which branch, in which surface. The output's job is to
be actionable without further investigation, since its reader is deciding
whether they have finished a change.

**Expects** that both implementations remain machine-readable by pattern
matching: a routing table recognisable as one, function definitions in a
conventional form, flag literals appearing as literals. This is a real
constraint on the two branches, not a free property, and it is the price of
having a guard at all under the no-dependencies rule.

**Failure contract.** A missing PowerShell branch — the state during
implementation, before the port lands — must produce a clear "not present"
result rather than a wall of spurious differences. The guard has to be usable
while the thing it guards is being built, or it will not be adopted.
Unparseable input is reported as unparseable, never silently treated as an
empty set: a guard that passes because it failed to read anything is the worst
possible outcome.

## Dependencies

- **[`powershell-shared-lib`](./powershell-shared-lib.md)** — supplies the
  function surface compared, and takes its naming rule from this feature's
  requirement.
- **[`powershell-command-surface`](./powershell-command-surface.md)** —
  supplies the subcommand and flag surfaces compared.
- **[`powershell-entry-point`](./powershell-entry-point.md)** — supplies the
  PowerShell routing table, and consumes the known-unported list to produce
  its "requires bash" message. The two must agree, which is itself worth
  checking.
- **`scripts/check-task-parity.sh`** — precedent rather than dependency. The
  new guard should look and behave like it, so the repository has one idea of
  what a guard is.
- **`docs/authoring-guide.md`** — carries the rule. Without it the guard is a
  script nobody runs.

## Open questions

- **When the guard runs.** The technical direction already records this as an
  outstanding question for `check-task-parity.sh`: the guard exists, documents
  reference it, but nothing invokes it and no document states who should. This
  feature inherits the problem and should not solve it separately — one
  protocol covering both guards is worth more than two conventions. Whether to
  settle it here or in a change to the authoring guide covering both is open.
- **Flag extraction reliability.** Subcommands and function definitions are
  easy to extract by pattern; flag literals are less so, since they may appear
  in help text, comments, and error messages as well as in parsing logic. An
  over-broad extraction produces false differences, which is the failure mode
  that gets a guard ignored. Whether flag comparison is worth including at the
  achievable precision is a judgement to make against the real source at
  planning time — the subcommand and function checks are worthwhile
  regardless.
