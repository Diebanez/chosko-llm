# PowerShell entry point

The chain that turns `chosko-llm` typed at a Windows prompt into a native
PowerShell subcommand: a retargeted `bin/chosko-llm.cmd`, a new
`bin/chosko-llm.ps1` proxy that dispatches, and the PowerShell twin of the
daily auto-upgrade hook. It is the seam where the Windows-native branch is
chosen over the bash one, and where the subcommands that were not ported are
handled honestly.

## Purpose

A ported command surface is unreachable without an entry point that routes to
it. This feature is what makes the port actually load on a machine with no
bash, and it is where the decision "which implementation runs" is taken —
once, in one place, rather than scattered across subcommands.

It also carries a piece of the product that is easy to lose in a port. The
bash proxy invokes the daily auto-upgrade hook before every dispatch; a
PowerShell proxy that omits it would leave Windows users silently never
auto-upgrading, with no error to notice. *Staying current* is a named
high-level feature in [`../product-design.md`](../product-design.md), and
losing half of it on one platform is a regression disguised as an omission.

## Scope and non-goals

**In scope.** `bin/chosko-llm.cmd` retargeted from locating git-bash to
launching PowerShell; `bin/chosko-llm.ps1` as the dispatching proxy; the
`CHOSKO_LLM_SHELL` override; the message shown for subcommands that exist in
bash but were not ported; and `scripts/auto-upgrade.ps1`.

**Out of scope, deliberately:**

- **Retiring `bin/chosko-llm.cmd`.** It stays, permanently. `.PS1` is not in
  the default `PATHEXT` and cmd.exe cannot execute PowerShell scripts at all,
  so the `.cmd` is the only file that makes the bare word `chosko-llm`
  resolve for every Windows caller. Replacing it with a `.ps1` would not be a
  modernisation; it would break cmd.exe users outright.
- **Changing `bin/chosko-llm`.** The bash proxy is untouched by this feature
  set. Unix behaviour must be bit-for-bit what it is today.
- **Dispatch logic in the `.cmd`.** It locates an interpreter and forwards.
  Subcommand routing lives in exactly one place per branch, and the `.cmd` is
  not one of them.
- **Porting `task-impl`.** Out of scope for the set; this feature owns only
  the message a user gets when they ask for it.
- **Changing the machine's PowerShell execution policy.** Covered below —
  this is a hard non-goal, not a preference.

## Architecture

Built on the stack recorded in
[`../technical-direction.md`](../technical-direction.md). The existing entry
chain — proxy dispatch, the `.cmd` shim, the auto-upgrade hook and its
skip rules — is documented in
[`../../.claude/context/cli-entry.md`](../../.claude/context/cli-entry.md)
and is not restated here.

### The chain

Typing `chosko-llm` on Windows resolves `chosko-llm.cmd` through `PATHEXT`,
from both cmd.exe and PowerShell. The `.cmd` selects an interpreter and
forwards; the selected proxy dispatches to a subcommand script. The bash proxy
and the PowerShell proxy are peers — neither wraps the other, and each routes
only within its own branch.

`chosko-llm.ps1` mirrors the bash proxy's structure exactly: verify the
managed clone exists and fail with the same guidance if not, take the first
argument as the subcommand, run the auto-upgrade hook guarded so it can never
block the user's command, then dispatch. Unknown subcommands exit 2, as today.

### Execution policy is bypassed per invocation, never set

Windows client machines default to a `Restricted` execution policy, under
which a `.ps1` file will not run. The `.cmd` therefore invokes PowerShell with
an execution-policy bypass **scoped to that single invocation**.

The rejected alternative matters more than the chosen one: instructing users
to run `Set-ExecutionPolicy`, or doing it for them during install, changes a
machine-wide security setting to make one tool work. That is the kind of
invasive footprint a port is most likely to acquire and hardest to walk back.
A per-invocation bypass leaves the user's policy exactly as it was.

The same reasoning shapes the bootstrap path, where `irm … | iex` sidesteps
policy naturally because piped script text is not a file — see
[`powershell-bootstrap`](./powershell-bootstrap.md).

### Which branch runs

PowerShell is the default on Windows. That is the point of the feature set:
the native branch must be the one users exercise, or it decays untested while
everyone stays on bash.

`CHOSKO_LLM_SHELL=bash` forces the bash branch. It exists for two concrete
reasons rather than as a general escape hatch: `task-impl` is only available
there, and comparing the two implementations on one machine is how parity gets
verified during implementation and after any CLI change. Absent or set to
anything else, PowerShell is used.

When the requested branch is unavailable — `CHOSKO_LLM_SHELL=bash` with no
bash present — the failure names what is missing and how to get it, rather
than silently falling back. A silent fallback would mean a user believing they
tested the bash branch when they tested the other.

### Unported subcommands degrade honestly

`task-impl` is a real subcommand of this product that this branch does not
implement. Routed through the PowerShell proxy it must produce a message
saying it requires bash and naming the way to run it — not "unknown
subcommand", which is what the proxy's default case would produce and which
would tell the user the feature does not exist.

The list of unported subcommands is a fact both branches must agree on, and
therefore something [`shell-parity-guard`](./shell-parity-guard.md) checks: a
subcommand added to bash and forgotten here becomes an "unknown subcommand"
for every Windows user, which is precisely the drift the guard exists to
catch.

### The auto-upgrade twin

`scripts/auto-upgrade.ps1` reproduces the bash hook's contract: fire
`chosko-llm upgrade` at most once per calendar day for users who have not
opted out, skip the subcommands the bash hook skips — `upgrade`, `help`, the
empty subcommand, `uninstall`, `version` — never recurse, honour
`CHOSKO_LLM_NO_AUTO_UPGRADE`, stamp `last_run` *before* upgrading so a failure
does not retry all day, and never abort the user's actual command whatever
goes wrong.

It shares the state file with the bash hook rather than keeping its own. This
is deliberate and is what makes mixed use coherent: upgrade once from
PowerShell and the bash branch knows it ran today. It is also why the state
file sits inside the fidelity invariant of
[`powershell-shared-lib`](./powershell-shared-lib.md) — a CRLF or a BOM in a
two-line `key=value` file would desynchronise the two hooks.

## Data and state

The entry point holds no state. It reads `CHOSKO_LLM_HOME` to locate the
managed clone and `CHOSKO_LLM_SHELL` to select a branch, and the auto-upgrade
twin reads and writes the shared `.auto-upgrade-state` file
(`enabled`, `last_run`) whose format is fixed by the bash branch.

One distribution fact shapes how changes to this feature reach users, and it
is a known rough edge rather than a new one. `upgrade` pulls the managed clone
and refreshes the proxy; the `.cmd` shim historically ships only via the
installer. So a change to the `.cmd` — including the retarget this feature
performs — reaches an existing Windows user only when they re-run the
installer. [`powershell-bootstrap`](./powershell-bootstrap.md) owns whether
that stays true, since it owns both the installer and the refresh path.

## Interfaces and contracts

**Exposes** to the user the same CLI surface as the bash proxy: the ten ported
subcommands plus `help` and `version`, `-v` / `--version` / `version` and
`""` / `-h` / `--help` / `help` aliases, exit 2 on an unknown subcommand.

**Expects** from [`powershell-command-surface`](./powershell-command-surface.md)
that a script exists per routed subcommand and accepts the forwarded arguments
with the subcommand stripped.

**The argument-forwarding contract** is where this feature is most likely to
fail quietly. Arguments cross two boundaries — cmd.exe to PowerShell, then
proxy to subcommand — and cmd.exe's `%*` forwarding is known to mangle
quoting, spaces, and the characters cmd.exe treats specially. Arguments must
arrive at the subcommand exactly as typed. Feature names are kebab-case and
survive anything, but paths passed to `export` and `--local` runs from
directories with spaces do not, and a corrupted path is a wrong file written
rather than an error raised.

**Exit codes propagate unchanged** through both hops to the calling shell.
Truncating them at either boundary breaks scripted use and would go unnoticed
in interactive testing.

**Failure contract.** Missing managed clone: the bash proxy's message and
guidance. Missing PowerShell: not a case worth handling, since 5.1 is present
on every supported Windows. Missing bash under an explicit
`CHOSKO_LLM_SHELL=bash`: a clear error, never a fallback. Auto-upgrade failure
of any kind: swallowed, exactly as in bash.

## Dependencies

- **[`powershell-shared-lib`](./powershell-shared-lib.md)** — the
  auto-upgrade twin needs the state helpers and the fidelity-preserving write
  path. The proxy itself deliberately depends on as little as possible,
  mirroring the bash proxy's rule that it must work before the managed clone
  has any scripts.
- **[`powershell-command-surface`](./powershell-command-surface.md)** — the
  dispatch targets. Neither feature is usable without the other.
- **[`powershell-bootstrap`](./powershell-bootstrap.md)** — places these files
  on the machine and owns how updates to them arrive.
- **Windows PowerShell 5.1** — the interpreter the `.cmd` launches.
- **cmd.exe** — not a dependency chosen but a fact of Windows: it owns
  `PATHEXT` resolution for bare command names, which is why the `.cmd` cannot
  be retired.

## Open questions

- **Whether `upgrade` should refresh the Windows shims.** Today `upgrade` does
  a git pull and refreshes the bash proxy only, so a fix to the `.cmd` or the
  `.ps1` proxy never reaches an existing Windows user without a manual
  installer re-run. Extending the refresh would close a real gap; leaving it
  keeps `upgrade` untouched and relies on documentation. Decided in
  [`powershell-bootstrap`](./powershell-bootstrap.md), which owns both sides.
- **Whether the proxy needs its own bash-branch delegation for `task-impl`.**
  A clear "use bash" message is the minimum; transparently re-invoking the
  bash branch when bash happens to be present would be friendlier, but makes
  the PowerShell proxy a conditional wrapper around the bash one and
  reintroduces the bash dependency through a side door for anyone who uses
  that path. Recommend the message; flagged rather than settled.
- **Whether `git.exe` without git-bash is sufficient.** Carried across this
  feature set until verified on real Windows hardware. It surfaces here
  through the auto-upgrade twin, which invokes `upgrade`, which invokes git.
