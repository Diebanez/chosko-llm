# PowerShell command surface

The ten native subcommand scripts — `scripts/cmd-<sub>.ps1` for `ls`, `add`,
`rm`, `update`, `upgrade`, `channel`, `show`, `export`, `help`, and `version`
— that let a Windows user install, inspect, and update features without a
bash shell. This is the bulk of the port: the feature that actually delivers
"use the tool without git-bash", with everything else in the set either
supporting it or protecting it.

## Purpose

`ls`, `add`, `rm`, and `update` are the whole of the zero-to-configured flow
and the whole of the `--local` deploy flow described in
[`../product-design.md`](../product-design.md) under *Feature catalogue and
deployment*; `upgrade` and `channel` are *Staying current*; `export` is
*Config export*. Together they are what a consuming teammate ever runs. Until
they exist natively, a Windows user has a bash-free entry point that
immediately hands them back to bash.

The subcommands are grouped into one feature because they share one substrate
— [`powershell-shared-lib`](./powershell-shared-lib.md) — one argument-parsing
convention, and one output contract. Splitting them per-command would produce
ten documents each saying "mirror the bash one", with the genuinely shared
decisions homeless. Which subcommands get ported in which order is a
sequencing question for `/task-add`, not an architectural one.

## Scope and non-goals

**In scope.** Native implementations of the ten subcommands above, at
behavioural parity with their bash counterparts: same flags, same resolution
rules, same output columns, same exit codes, same refusals.

**Out of scope, deliberately:**

- **`task-impl`.** The external-LLM orchestrator is the largest and most
  stateful script in the repository, it drives `aider` and Ollama, and it is
  an authoring flow rather than a consumption one. On Windows it stays on the
  bash path. [`powershell-entry-point`](./powershell-entry-point.md) owns
  saying so clearly at the moment a user asks for it, rather than failing with
  "unknown subcommand".
- **`install` and `uninstall`.** They are not proxy subcommands in the same
  sense — they must run before the managed clone exists. They belong to
  [`powershell-bootstrap`](./powershell-bootstrap.md).
- **New flags, new behaviour, new output.** Anything that would be an
  improvement is out until it exists on both branches. A flag that works only
  in PowerShell is drift.
- **Interactive rewrites.** `uninstall`'s confirmation prompts stay prompts;
  no PowerShell-native `ShouldProcess` / `-WhatIf` support, however idiomatic,
  because the bash branch has no equivalent.

## Architecture

Built on the stack recorded in
[`../technical-direction.md`](../technical-direction.md) and on
[`powershell-shared-lib`](./powershell-shared-lib.md), which every script here
loads first and through which every shared-file write passes. The existing
bash behaviour each script mirrors is documented per-command in the context
layer — [`../../.claude/context/cmd-ls.md`](../../.claude/context/cmd-ls.md)
and its siblings, indexed in
[`../../.claude/context/INDEX.md`](../../.claude/context/INDEX.md). This
document does not restate any of it.

The architecture is deliberately boring: one script per subcommand, no shared
dispatcher of its own (that is the proxy's job), no cross-subcommand imports
beyond the library. The design content is in the four contracts below, which
are where a mirror implementation silently stops being a mirror.

### Argument parsing stays bash-shaped

The CLI's flags are `--all`, `--installed`, `--available`, `--local`,
`--global`, `--enable-auto`, `--disable-auto`, `--list`, `-y` / `--yes`, and
the `kind:name` prefixed feature specs. PowerShell's parameter binder wants
`-All`, `-Installed`, case-insensitively, with prefix abbreviation — a
different surface, silently accepted.

The scripts therefore parse the raw argument array themselves rather than
declaring `param()` blocks with switch parameters. This is the single most
likely place for accidental divergence, because PowerShell's convenience is
what causes it: a user typing `chosko-llm ls -inst` must fail on both
branches or neither, and a script that lets the binder do the work will
accept it on one.

Scope resolution keeps its order-agnostic contract — `--local` / `--global`
may appear anywhere in the argument list, both together is refused, and local
scope requires a `CLAUDE.md` in the current directory before it will target a
project's `.claude/`.

### Output is text, not objects

PowerShell's instinct is to emit objects and let the host format them. Every
script here emits preformatted text on stdout, matching the bash column
layout, header row, alignment, and TTY-only footer hints.

The reason is not aesthetic. `ls` output is read by humans comparing installed
against available versions at a glance, and the column semantics carry meaning
that the palette reinforces — status colours and kind colours are separate
vocabularies that deliberately never collide. An object stream reformatted by
`Format-Table` on a narrow console truncates columns and drops the alignment
that comparison depends on. It would also make the parity guard's job
impossible, since there would be nothing textual to compare.

### Failure and exit codes are part of the contract

Each script keeps its bash counterpart's exit codes and its refusals: `add`
refuses a feature already installed, `add` and `update` refuse a source file
whose frontmatter lacks a non-empty `name` or `version`, both refuse a
statusline in local scope and a hook in global scope, and an ambiguous bare
feature name is an error rather than a guess.

`upgrade` and `channel` shell out to `git` and must propagate its failures
rather than swallowing them — PowerShell does not set a failing exit status
for native executables the way `set -e` does for bash, so this is an explicit
obligation here rather than something the shell provides.

### Post-install prompts are reproduced verbatim

Installing a statusline or a hook prints a copy-pasteable prompt telling the
user to have a Claude Code session merge the relevant key into
`settings.json` — the product's deliberate answer to editing JSON without
`jq`. The hook prompt wires the repo-relative
`$CLAUDE_PROJECT_DIR/.claude/hooks/<name>.sh` path rather than the absolute
install path, because `settings.json` is committed and travels to other
machines and to cloud containers.

That path must not acquire a Windows drive letter or backslashes on its way
through the PowerShell branch. A prompt generated on Windows and pasted into a
repository that a Linux cloud agent later clones has to still work. This is
the fidelity invariant of
[`powershell-shared-lib`](./powershell-shared-lib.md) reaching into
user-visible text rather than only into file contents.

## Data and state

The scripts introduce no state. They read and write what already exists:
feature files in the managed clone, installed artifacts under `$CLAUDE_HOME`,
managed sections in the target `CLAUDE.md`, the auto-upgrade state file, and
export artifacts under `$CHOSKO_LLM_EXPORT_DIR` (defaulting to
`~/claude-exports`).

Two consequences of the filesystem being the state carry over unchanged.
Installed and available versions are both read from frontmatter at call time,
so there is nothing to keep in sync and nothing to invalidate. And install is
copy-based and never prunes, which is why the `replaces:` kind-migration
helpers exist — a feature that changes kind would otherwise leave a stale
artifact beside the new one. That migration must work identically across
branches, including a migration begun on one and completed on the other.

`export` is the only subcommand that writes outside the two managed trees. Its
output must be readable on the machine that receives it, which is a further
instance of the encoding invariant rather than a separate rule.

## Interfaces and contracts

**Exposes** the CLI surface documented in `docs/cli-help.txt` — the same one
the bash branch exposes. That file is shared, not duplicated, so help text
cannot drift by construction.

**Expects** from [`powershell-entry-point`](./powershell-entry-point.md): to
be invoked with the subcommand already stripped and the remaining arguments
forwarded intact, including arguments containing spaces and quotes. It expects
from [`powershell-shared-lib`](./powershell-shared-lib.md) the full helper
surface, and it must not reach around it to write a shared file.

**Guarantees to the user**, and this is the contract that matters most: a
command run from PowerShell and the same command run from git-bash on the same
machine leave the same bytes on disk. Not equivalent state — the same state.
Switching branches mid-session, or between sessions, is supported and needs no
migration.

**Failure contract.** Invalid input fails before any file is touched;
validation precedes copying. A partially completed multi-feature operation
(`--all`) reports what succeeded rather than claiming the whole run failed,
matching bash. Missing `git` surfaces as a clear error naming git, not as a
PowerShell command-not-found trace.

## Dependencies

- **[`powershell-shared-lib`](./powershell-shared-lib.md)** — every helper
  these scripts use, and the fidelity invariants they must not bypass. Hard
  dependency: nothing here can be built first.
- **[`powershell-entry-point`](./powershell-entry-point.md)** — routes
  subcommands here. Buildable in either order, but neither is usable alone.
- **`git.exe`** — required by `upgrade` and `channel`, which are git
  operations on the managed clone. Not bundled and not optional; see
  [`powershell-bootstrap`](./powershell-bootstrap.md).
- **`docs/cli-help.txt`** — shared with the bash branch, printed by `help`.
- **The context layer** — `.claude/context/cmd-*.md` is the specification each
  script mirrors, and the cheapest correct description of the behaviour to
  reproduce.

## Open questions

- **How `export`'s zip is produced.** The bash branch's approach and
  PowerShell's `Compress-Archive` may not agree on entry paths or on
  separators inside the archive, and an archive produced on Windows should
  unpack correctly on a Unix machine. Blocks `export` only.
- **Whether `--all` ordering is observable.** Both branches walk directories
  to enumerate features, and directory enumeration order is not guaranteed to
  match between the two. If the order shows up in output, it is part of parity
  and must be sorted explicitly; if it never does, this is nothing. Cheap to
  settle by inspection at planning time.
- **Whether `git.exe` without git-bash is sufficient.** Carried across this
  feature set until verified on real Windows hardware. `upgrade` and `channel`
  are where it would surface first, since they are the subcommands that
  actually invoke git.
