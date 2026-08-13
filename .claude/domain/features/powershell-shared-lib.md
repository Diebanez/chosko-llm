# PowerShell shared library

`scripts/lib.ps1` is the PowerShell counterpart of `scripts/lib.sh` — the
helper layer every native subcommand builds on. It is the foundation of the
Windows-native branch: nothing else in the PowerShell port can be written
until the helper surface exists, and every fidelity guarantee the port makes
about the files it writes is enforced here rather than in the subcommands.

## Purpose

The product's Windows story today is "install Git for Windows and work from
git-bash". The goal recorded for this feature set is to remove bash as a
runtime requirement on Windows entirely, so the tool can be used from the
PowerShell that every Windows 10/11 machine already has.

`lib.sh` is where that requirement concentrates. It is sourced by every
`cmd-*.sh` and carries roughly fifty helpers — logging, colour, scope
resolution, frontmatter parsing, path assembly, claude-md section injection,
statusline and hook prompts, kind migration, validation, and auto-upgrade
state. A native branch that cannot reproduce this surface cannot host a
single subcommand. See [`../../.claude/context/shared-lib.md`](../../.claude/context/shared-lib.md)
for the existing surface in full; this document does not restate it.

The technical direction records native Windows support as an explicitly open
decision, naming ongoing maintenance of a second implementation as the cost
that matters — see
[`../technical-direction.md`](../technical-direction.md). This feature set
takes the PowerShell route and leaves WSL unexplored; the maintenance cost is
answered by [`shell-parity-guard`](./shell-parity-guard.md), not by this
document.

## Scope and non-goals

**In scope.** A PowerShell module or dot-sourced script exposing the helper
surface the ported subcommands need, at behavioural parity with `lib.sh`, and
the on-disk fidelity invariants described below.

**Out of scope, deliberately:**

- **Helpers only `task-impl` needs.** `lib-task-external.sh` is not ported;
  the backlog-parsing and guard helpers beneath the external-LLM orchestrator
  stay bash-only. `lib.ps1` mirrors `lib.sh` and nothing else.
- **Improving on `lib.sh`.** Where a helper's bash behaviour is awkward, the
  PowerShell twin reproduces the awkwardness. Divergence is the failure mode
  this whole feature set is built to prevent; a better design applied to one
  branch only is drift wearing a nicer name.
- **Cross-platform PowerShell.** The target is Windows PowerShell 5.1 on
  Windows. Running `lib.ps1` under pwsh on macOS or Linux is not a goal and
  is not tested — Unix keeps the bash branch.
- **A public extension API.** `lib.ps1` serves this repository's subcommands.
  It is not a module for third parties to build on, and nothing about its
  surface is promised to anyone outside the repo.

## Architecture

Built on the stack recorded in
[`../technical-direction.md`](../technical-direction.md), extended with one
addition that document explicitly leaves open: Windows PowerShell 5.1 as a
second host language on Windows only. The no-dependency rule carries across
unchanged — 5.1 ships in the box, so requiring it adds no install, and no
module from the PowerShell Gallery may be taken as a dependency.

The library is organised into the same helper groups `lib.sh` uses, so a
reader who knows one branch can navigate the other. The interesting design
content is not the grouping — it is the three concerns below, which have no
counterpart in the bash branch because bash gets them right by default.

### Language floor: Windows PowerShell 5.1

5.1 is preinstalled on every Windows 10 and 11 machine, which is what makes
"no installs beyond git" a promise the product can keep. The cost is a dated
language: no ternary operator, no null-coalescing, different JSON handling,
and — the consequential one — output encoding defaults that are actively
wrong for this product.

There are no conditional branches on PowerShell version. Code is written
against 5.1 and runs unchanged under 7 for anyone who has it. Two runtimes
tested is not worth the language conveniences.

### On-disk fidelity

This is the sharpest risk in the entire port, and it is the reason `lib.ps1`
is a separately designed feature rather than an implementation detail of the
subcommands.

Both branches write the same files on the same machine. A user may install
from PowerShell and later update from git-bash, or the reverse; the managed
clone, `$CLAUDE_HOME`, the target `CLAUDE.md`, and the auto-upgrade state
file are shared, not per-branch. So it is not enough that the two branches
behave equivalently — the bytes they write must be interchangeable.

Windows PowerShell 5.1 defaults work against this in three ways:

| Concern | 5.1 default | Required behaviour |
| --- | --- | --- |
| `Out-File` encoding | UTF-16LE | UTF-8 |
| `Set-Content -Encoding UTF8` | UTF-8 **with BOM** | UTF-8 without BOM |
| Line endings | CRLF | LF |

The failure this prevents is concrete. claude-md artifacts are injected into
`CLAUDE.md` between `<!-- chosko-llm:<name>:begin v<version> -->` and `:end`
markers, and the bash branch locates those markers with `awk`/`sed`. A BOM
ahead of the first marker, or CRLF inside one, and the bash-side
`inject_section` / `remove_section` stop matching — leaving a user with a
section that can be written but never removed, or duplicated on the next
install.

Every write in `lib.ps1` therefore goes through a single internal file-writing
helper that fixes encoding and line endings in one place. No subcommand calls
`Out-File`, `Set-Content`, or `>` directly. This is the same discipline the
bash branch already applies to path assembly, where the path helpers are the
only place `$CHOSKO_LLM_HOME` and `$CLAUDE_HOME` concatenate with subpaths.

The invariant is stated positively so it can be checked: **any file both
branches may write is UTF-8 without BOM and LF-terminated, regardless of
which branch wrote it.**

### Console output

`lib.sh` writes log lines to stderr and holds a stdout colour palette
(`C_GREEN`, `C_YELLOW`, `C_CYAN`, `C_BLUE`, `C_MAGENTA`, `C_DIM`, `C_BOLD`,
`C_RESET`) whose members carry assigned meanings across `ls` and `show` — a
status vocabulary and a kind vocabulary that deliberately do not collide.
Reproducing those assignments is straightforward; reproducing the *mechanism*
is not, and the choice is recorded as an open question below.

Two properties are fixed regardless of that choice: the stream split must be
preserved, because callers may redirect stdout while still wanting logs; and
`NO_COLOR` must be honoured, along with the "is this a terminal" test that
`lib.sh` performs with `[ -t 1 ]` / `[ -t 2 ]`.

### Path and environment semantics

`CHOSKO_LLM_HOME`, `CLAUDE_HOME`, `CHOSKO_LLM_SCOPE`, and
`CHOSKO_LLM_EXPORT_DIR` keep their names, their defaults, and their override
precedence. A user who has set them for the bash branch finds them working
identically here.

Two Windows-specific wrinkles fall to this layer rather than to the callers:
paths must round-trip between the forward-slash form the shared files use and
the native form Windows APIs expect, and `$HOME` is not guaranteed to be set
on Windows the way it is under bash — the fallback chain to `$env:USERPROFILE`
lives in one place here.

## Data and state

`lib.ps1` owns no state of its own. It reads and writes state that already
exists and whose shape is fixed by the bash branch:

- **`$CHOSKO_LLM_HOME/.auto-upgrade-state`** — gitignored, two keys
  (`enabled`, `last_run`) in `key=value` form. A missing file or key means
  enabled. `last_run` is a calendar date. Both branches read and write it, so
  it is squarely inside the fidelity invariant.
- **Feature frontmatter** — the first `---` block of a feature file, five
  recognised keys (`name`, `version`, `type`, `description`, `replaces`) plus
  the hook-only `event:` and `matcher:`. Read-only from this layer.
- **The target `CLAUDE.md`** — read and rewritten by section injection and
  removal, delimited by the managed-section markers. Its path differs by
  scope, and that resolution is a helper rather than a hardcoded path.
- **Installed artifacts under `$CLAUDE_HOME`** — discovered by walking
  directories. The filesystem is the state; there is no index and no
  lockfile, exactly as recorded in the product design.

Nothing here introduces a new file, a cache, or a per-branch marker. A
PowerShell-specific state file would itself be a divergence.

## Interfaces and contracts

**Exposes** the `lib.sh` helper groups, named so that the correspondence is
mechanically checkable by
[`shell-parity-guard`](./shell-parity-guard.md): logging and `die`, the stdout
colour palette, scope resolution and its scope-restricted-kind rules,
frontmatter parsing and field reads, source and installed path assembly,
export directory resolution, claude-md section injection and removal,
statusline and hook wiring prompts, feature-kind resolution and prefix
parsing, the `replaces:` kind-migration helpers, source validation, and
auto-upgrade state access.

The mapping between a bash function name and its PowerShell counterpart must
be derivable by rule rather than by lookup table, since the parity guard has
to compute it. Choosing that rule — whether PowerShell naming conventions are
adopted with a mapping, or bash names are kept verbatim at the cost of
idiom — is settled in `shell-parity-guard`, which is the consumer of the
answer.

**Expects** from its callers: that it is loaded once at the top of a
subcommand before any other work, mirroring `source lib.sh`; and that no
caller bypasses it to touch a shared file or assemble a managed path directly.

**Failure contract.** `die` writes an error and exits 1, matching `lib.sh`
exactly — subcommand-specific exit codes stay in subcommands. Two failures
belong here specifically: a source file missing a non-empty `version` or
`name` is refused before any copy, and a hook missing `event:` is refused
rather than half-installed. A malformed `replaces:` spec warns and no-ops
rather than failing the run, as it does today.

Exit codes are part of the interface and part of parity. The proxy propagates
them to the shell, and scripting users depend on them.

## Dependencies

- **No other feature in this set.** This is the foundation; every other
  PowerShell feature depends on it.
- **Windows PowerShell 5.1** — the host language. Preinstalled on Windows
  10/11, which is the entire reason it was chosen over pwsh 7.
- **`git.exe`** — not used directly by this layer, but the branch it belongs
  to cannot function without it. Recorded in
  [`powershell-bootstrap`](./powershell-bootstrap.md).
- **`scripts/lib.sh`** — not a runtime dependency but a specification one.
  It is the authority on what every helper does; where the two disagree, bash
  is correct by definition and the PowerShell side is the bug.

## Open questions

- **Colour mechanism on the 5.1 console.** ANSI escapes require virtual
  terminal processing, available from Windows 10 1511 but not enabled in every
  host; `Write-Host -ForegroundColor` always works but writes directly to the
  host, which interacts badly with redirection and with the stdout/stderr
  split this library must preserve. The choice affects how faithfully the
  palette parity can be claimed and how `NO_COLOR` is implemented. Blocks the
  colour helpers only; every other helper can be built first.
- **Function naming rule.** Whether to keep `lib.sh` names verbatim for
  mechanical checkability, or adopt `Verb-Noun` PowerShell convention with a
  derivable mapping. Blocks the parity guard's comparison logic more than it
  blocks this library. Resolved jointly with
  [`shell-parity-guard`](./shell-parity-guard.md).
- **Whether `git.exe` without git-bash is sufficient.** Some git porcelain has
  historically shipped as shell scripts, which would mean a bash dependency
  survives underneath a bash-free CLI. This does not block `lib.ps1`, which
  invokes no git, but it is the single finding that could undermine the
  premise of the whole feature set, so it is carried in every document until
  verified on real Windows hardware.
