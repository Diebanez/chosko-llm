# PowerShell bootstrap

`install.ps1` and `uninstall.ps1`: getting `chosko-llm` onto a Windows machine
that has never had it, and taking it off again, without ever opening a bash
shell. It is the Windows counterpart of `install.sh` — the step that must work
before anything else in the product exists on the machine.

## Purpose

This is the feature the whole set is judged by. A native command surface and a
native entry point still leave a Windows user stuck if the only way to install
them is to obtain Git for Windows, find git-bash in the Start menu, and run a
shell script from it. *Machine bootstrap* in
[`../product-design.md`](../product-design.md) promises that a machine goes
from nothing to fully configured in one command; on Windows that promise is
currently kept only for people who already live in bash.

The bar is the one Unix already meets: a single line pasted into the shell the
user already has open. On Unix that is `curl … | bash`; here it is
`irm … | iex` in the PowerShell that ships with Windows.

## Scope and non-goals

**In scope.** `install.ps1` and `uninstall.ps1` at behavioural parity with
their bash counterparts; git detection and an offer to install it via winget;
the Windows PATH story; the pipe-invocation guard; and the decision about how
updates to the Windows entry files reach existing users.

**Out of scope, deliberately:**

- **Removing `git.exe` as a requirement.** The distribution model *is* a git
  clone — `install` clones, `upgrade` pulls, `channel` checks out. Removing
  git would mean redesigning the product, not porting it. The promise this
  feature set makes is "no bash", not "no prerequisites", and that distinction
  belongs in the user-facing documentation rather than being quietly blurred.
- **Installing anything except git.** No PowerShell 7, no Claude Code, no
  package manager. Git is bootstrapped because the tool cannot function
  without it and because it is one winget command away; that reasoning does
  not generalise.
- **Modifying the system PATH or the machine execution policy.** Both are
  machine-wide changes made to accommodate one tool. The user PATH is the
  furthest this goes, and execution policy is not touched at all — see
  [`powershell-entry-point`](./powershell-entry-point.md).
- **Replacing `install.sh`.** It stays and remains authoritative on Unix. On
  Windows both installers must produce the same result, so a user who
  installed from git-bash yesterday and runs `install.ps1` today gets an
  update, not a second parallel installation.

## Architecture

Built on the stack recorded in
[`../technical-direction.md`](../technical-direction.md). The existing
bootstrap — its clone branches, backup policy, auto-upgrade state
initialisation, version banner, and PATH checks — is documented in
[`../../.claude/context/cli-entry.md`](../../.claude/context/cli-entry.md) and
mirrored rather than redesigned.

### The same steps, in the same order

`install.ps1` performs `install.sh`'s sequence: resolve the managed clone and
bin directory from `CHOSKO_LLM_HOME` and `BIN_DIR` with the same defaults;
clone or fast-forward the managed clone; place the proxy files in the bin
directory, backing up anything already there under a timestamped name;
initialise the auto-upgrade state without clobbering an existing preference;
report the resolved version through the same helper the CLI uses, so the
version string cannot drift from `chosko-llm --version`; and finish with a
PATH check and a next-step hint.

Idempotence is part of the parity. Re-running on a populated managed clone
fast-forwards rather than failing, exactly as bash does, and re-running is the
supported way to repair a broken install.

### The pipe-invocation guard

`install.sh` already solves this for `curl | bash`: piped into an interpreter,
there is no script file, so the usual "where am I" reference is unset and the
installer must fall back to cloning a known repository URL rather than
deriving one from a local working copy. The bash version guards its script-path
reference explicitly and leaves it empty, which selects the clone-from-URL
branch.

PowerShell under `irm … | iex` has the identical property for the identical
reason, and needs the identical three-branch resolution: an existing managed
clone is pulled; a developer working copy supplies its own origin URL; and a
pipe invocation, having neither, clones the default repository URL with an
override available.

This symmetry is worth naming because it is the kind of thing a port gets
wrong by not knowing the bash version had a reason. The consequence of missing
it is an installer that works when tested from a checkout and fails for every
real first-time user.

### Git detection and the winget offer

Git is detected before anything else, since every subsequent step needs it.
When it is missing, the installer offers to install it via winget, proceeding
only on explicit confirmation. Confirmation is the whole design: installing
third-party software unprompted is exactly the invasive footprint this port is
trying not to acquire, and the product's culture is to gate and let the user
decide.

Two fallbacks matter. When winget is absent — older or unpatched Windows — the
installer prints the download link and the exact command to run, then stops.
And when the user declines, that is a normal outcome reported clearly, not an
error.

Git and git-bash are separable in practice: git installed via winget or scoop
provides `git.exe` without the user ever opening the bash shell it ships
alongside. That separation is what lets this feature set claim "no bash" while
still requiring git. Whether it holds all the way down is the set's one
unverified premise — see open questions.

### PATH on Windows

The bash installer warns that the bin directory must be on the *Windows* PATH,
not merely the MSYS one, and prints the native path form with the manual
System Properties route. That warning exists because `install.sh` cannot
sensibly modify the Windows environment from inside MSYS.

`install.ps1` has no such excuse — it is running natively and can read and
write the user's PATH environment variable. The design decision is to offer
that, with confirmation, and to fall back to the existing printed instructions
when declined. The user scope is the ceiling: the machine PATH is never
touched, and a change to the user PATH is reversible from the same place the
user would look for it.

A newly added PATH entry does not apply to the already-open session, and the
installer says so explicitly rather than leaving the user to conclude the
install failed when the next command is not found.

### Uninstall

`uninstall.ps1` mirrors `uninstall.sh`: confirm up front before removing
anything, remove the proxy files, optionally remove installed features by
intersecting the managed clone's feature listings against `$CLAUDE_HOME` so
user-authored files are never touched, and optionally remove the managed clone
itself. `-y` / `--yes` auto-confirms.

The intersection rule is the safety property worth stating: uninstall deletes
only what this product could have installed. That must not become more
aggressive in the port, and a PowerShell recursive delete is an easy way to
make it so.

If a Windows PATH entry was added by `install.ps1`, uninstall offers to remove
it — leaving a dangling PATH entry behind would be a change the user cannot
easily attribute later.

## Data and state

The bootstrap creates the state everything else assumes: the managed clone at
`CHOSKO_LLM_HOME` (a git clone, its remote the state that makes `upgrade` and
`channel` work), the proxy files in `BIN_DIR`, timestamped backups of anything
replaced, and `.auto-upgrade-state` seeded as enabled unless a preference
already exists.

The user PATH environment variable becomes state this feature writes, which is
new — no other part of the product modifies the user's environment. It is the
reason uninstall must be able to reverse it.

Everything written here that either branch may read falls under the fidelity
invariant of [`powershell-shared-lib`](./powershell-shared-lib.md): UTF-8
without BOM, LF-terminated. The two-line auto-upgrade state file is the
concrete case — seeded by whichever installer ran, read by both hooks.

## Interfaces and contracts

**Exposes** two entry points: `install.ps1`, taking no arguments and reading
`CHOSKO_LLM_HOME`, `BIN_DIR`, and `REPO_URL` from the environment with the
same defaults as bash; and `uninstall.ps1`, interactive with `-y` / `--yes`.
`uninstall.ps1` must also be reachable as `chosko-llm uninstall` through the
PowerShell proxy, matching the bash branch's thin-wrapper arrangement, so
there is a single teardown implementation per branch.

**Guarantees** that the on-disk result is indistinguishable from what
`install.sh` produces on the same machine. This is what makes the two branches
interchangeable rather than merely coexistent, and it is checkable: install
from one, verify with the other.

**Failure contract.** Git missing and winget unavailable, or the offer
declined: clear message naming what is needed, non-zero exit, nothing written.
Clone failure: git's own error surfaced, not swallowed. An existing file in
the bin directory: backed up, never overwritten in place. A managed clone that
exists but is not a git repository, or whose fast-forward fails: reported
rather than force-resolved — destroying a user's local state to make an
install succeed is worse than failing.

## Dependencies

- **[`powershell-entry-point`](./powershell-entry-point.md)** — supplies the
  files this feature places and the branch-selection behaviour it enables.
  This feature is the last of the three to become useful.
- **[`powershell-shared-lib`](./powershell-shared-lib.md)** — sourced in
  isolation to resolve the version string, mirroring the narrow exception
  `install.sh` already makes for the same purpose. Otherwise the installer
  stands alone, since it must run before the managed clone exists.
- **`git.exe`** — hard requirement, bootstrapped via winget on confirmation.
  The one dependency this feature is allowed to install.
- **winget** — optional. Absent, the installer degrades to printed
  instructions.
- **Windows PowerShell 5.1** — present on every supported Windows.

## Open questions

- **Whether `upgrade` should refresh the Windows entry files.** Inherited from
  [`powershell-entry-point`](./powershell-entry-point.md) and owned here.
  Today the shim ships only through the installer, so a fix to the `.cmd` or
  the `.ps1` proxy never reaches an existing Windows user without a manual
  re-run — a gap that the retarget in this feature set makes newly relevant,
  since it changes the `.cmd` for everyone. Options: extend `upgrade` to
  re-copy the entry files when they are present in the bin directory; or leave
  `upgrade` untouched and document the re-run. The first closes a real bug at
  the cost of touching a shared subcommand that the bash branch also uses;
  the second is free and inadequate. Recommend the first, but it changes bash
  behaviour and so is not settled unilaterally here.
- **Whether `git.exe` without git-bash is sufficient.** The premise of the
  entire feature set. Some git porcelain has historically shipped as shell
  scripts, which would mean a bash dependency survives beneath a bash-free
  CLI. Only the subcommands invoking git are affected — `upgrade` and
  `channel` — and only for whichever operations turn out to be scripts. Must
  be verified empirically on a Windows machine with git installed and git-bash
  absent, before the set's user-facing claim is written down.
- **Whether the winget offer belongs in an unattended install.** `irm … | iex`
  is sometimes run from provisioning scripts where no one is present to
  confirm. A non-interactive mode would need a flag, and a flag on an
  installer invoked by pipe is awkward to pass. Not blocking; nothing in the
  product is scripted this way today.
