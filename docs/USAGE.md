# Using ion-shim

This is a practical, step-by-step guide to setting up and using the shim.
For the design rationale behind each of these behaviors, see the main
[README](../README.md); this document is about *doing*, not *why*.

**Scope note:** this repo ships two things — the shim binary
(`src/ion_shim.cto`) and a small `ion` CLI (`src/ion.cto`) implementing
`install`/`use`/`shim add`/`shim remove`/`shim list`/`pin`. What `ion`
does **not** do yet is fetch anything — `ion install` takes an
already-obtained local file, not a URL or package name to download.
Every section below shows both the `ion` command and, underneath it,
the raw filesystem operations it performs — useful for understanding
what's actually happening, or for scripting around `ion` if you'd
rather not depend on it.

## 1. Prerequisites

You need the `certo` compiler on `PATH` to build the shim. It's not
packaged anywhere conventional (no apt/choco/winget entry) — build it
from source:

```bash
cargo install --git https://github.com/rjreeves/Certo certo
```

(Note: `certo` is a Cargo workspace with several binary-producing
crates, so the package name `certo` must be passed explicitly — `--bin
certo` alone is ambiguous and fails.)

## 2. Build the shim and the installer

```bash
certo src/ion_shim.cto -o ion_shim.exe
certo src/ion.cto -o ion.exe
```

`ion_shim.exe` is generic — it doesn't know which tool it's fronting
until you copy it under a specific name (step 5). `ion.exe` is the CLI
that does that copying and the rest of the bookkeeping for you; keep it
wherever's convenient on your own `PATH` (it isn't part of the `Ion\`
layout itself, unlike the shim).

## 3. Set up the directory layout

The shim expects this structure under `%LOCALAPPDATA%\Ion` (or wherever
`ION_HOME` points, if you set that environment variable):

```
%LOCALAPPDATA%\Ion\
    shim\       <- the master ion_shim.exe build lives here, never on PATH
    bin\        <- per-tool copies of shim\ion_shim.exe; the only PATH entry needed
    shims\      <- one <name>.toml descriptor per shimmed tool
    packages\   <- actual installed binaries, one folder per version
```

Create the four folders once:

```powershell
New-Item -ItemType Directory -Force "$env:LOCALAPPDATA\Ion\shim"
New-Item -ItemType Directory -Force "$env:LOCALAPPDATA\Ion\bin"
New-Item -ItemType Directory -Force "$env:LOCALAPPDATA\Ion\shims"
New-Item -ItemType Directory -Force "$env:LOCALAPPDATA\Ion\packages"
```

Copy the binary you just built into its permanent home:

```powershell
Copy-Item ion_shim.exe "$env:LOCALAPPDATA\Ion\shim\ion_shim.exe"
```

Keeping one master copy here — separate from `bin\`, and not wherever
it happened to be built — means shimming a new tool later is always a
copy from `shim\ion_shim.exe`, never a recompile from Certo source.

Add `%LOCALAPPDATA%\Ion\bin` to your `PATH` — this is the **only** PATH
change ever required, no matter how many tools you shim later.
`shim\` is deliberately never added to `PATH`.

## 4. Install a real version of a tool

You already need the actual binary on disk somewhere — `ion install`
places it, it doesn't fetch it:

```bash
ion install certo@1.8.0 C:\path\to\certo-1.8.0.exe certo.exe
```

This copies (binary-safe — see the code walkthrough for why that
matters) the given file to `packages\certo\1.8.0\certo.exe`, creating
the version folder if needed. The trailing `certo.exe` names the file
inside that folder; if omitted, it defaults to the source file's own
name. Equivalent by hand:

```powershell
New-Item -ItemType Directory -Force "$env:LOCALAPPDATA\Ion\packages\certo\1.8.0"
Copy-Item "C:\path\to\certo-1.8.0.exe" "$env:LOCALAPPDATA\Ion\packages\certo\1.8.0\certo.exe"
```

## 5. Shim the tool

```bash
ion shim add certo certo@1.8.0
```

This copies `shim\ion_shim.exe` to `bin\certo.exe` (always re-copying,
even if already present — cheap, and picks up a rebuilt shim
automatically) and writes `shims\certo.toml`. An optional fourth
argument names the command inside the version folder if it isn't
`<name>.exe` (e.g. `ion shim add certo certo@1.8.0 certo-cli.exe`).
Equivalent by hand:

```powershell
Copy-Item "$env:LOCALAPPDATA\Ion\shim\ion_shim.exe" "$env:LOCALAPPDATA\Ion\bin\certo.exe"
```

```toml
# shims\certo.toml
package = "certo"
command = "certo.exe"
version = "1.8.0"
```

See [examples/shims/certo.toml](../examples/shims/certo.toml) for a
working example. Format rules: flat `key = "value"` lines only (no
nesting), `#` starts a full-line comment, blank lines are ignored,
quotes around the value are optional.

That's it — `certo --version` typed anywhere now runs through the shim
to the real `1.8.0` binary. `ion shim list` prints every currently
shimmed name; `ion shim remove certo` deletes both `bin\certo.exe` and
`shims\certo.toml` (leaving installed packages alone, since another
name might still reference the same package/version).

## 6. Switch versions

Install the new version the same way as step 4, then:

```bash
ion use certo@1.9.0
```

This rewrites `version` in **every** `shims\*.toml` whose `package`
matches — one command, whether `certo` is fronted by one name or, per
the toolchain design, several. Equivalent by hand: edit just the
`version` line in `shims\certo.toml`:

```toml
package = "certo"
command = "certo.exe"
version = "1.9.0"
```

No rebuild, no PATH change, no re-copying the shim. This is the entire
value of the design — switching is a one-line text edit, whether you
make it yourself or `ion use` makes it for you.

## 7. Pin a version per-project

```bash
cd my-project
ion pin certo@1.7.0
```

Writes (or updates, preserving every other line including comments) an
`ion.toml` in the current directory to override the global version for
just that directory tree. Pass a directory as a third argument to pin
somewhere other than the current one: `ion pin certo@1.7.0 C:\path\to\project`.
By hand, that's just:

```toml
# my-project/ion.toml
certo = "1.7.0"
```

Any number of tools can share one file:

```toml
certo = "1.7.0"
flux  = "1.2.0"
```

Running `certo` from inside `my-project` (or any subdirectory under
it) now uses `1.7.0` regardless of the global setting — the shim walks
up from the current directory looking for the nearest `ion.toml`. See
the README's "Project-level pinning" and "Nested projects" sections for
the exact boundary rules (in short: the nearest file found wins
entirely, it doesn't merge with parent files).

**The tool must already be shimmed globally before you can pin it.**
`ion.toml` only overrides which *version* runs — it can't invent a
`package`/`command` mapping that doesn't exist yet in `shims\`. If you
pin a tool that was never shimmed, you'll get a specific error telling
you so (see the error reference below).

## 8. Set up a version alias (`latest`, `lts`, ...)

Create a directory junction (not a symlink — junctions don't need admin
rights):

```powershell
New-Item -ItemType Junction `
  -Path "$env:LOCALAPPDATA\Ion\packages\certo\latest" `
  -Target "$env:LOCALAPPDATA\Ion\packages\certo\1.9.0"
```

Now `version = "latest"` works in either a descriptor or an `ion.toml`
pin, exactly like any real version string — the shim doesn't know or
care that it's a junction. Repoint the junction whenever a newer
version is installed to make it track forward, or leave it alone to
keep it frozen. See the README for the reproducibility caveat about
using a floating alias in a checked-in project pin.

## 9. Shim a multi-binary release (a "toolchain")

If a release exposes several binaries — Certo's ships `certo`,
`certo-fmt`, `certo-lsp`, and more — place them all in the same version
folder and write one descriptor per name, sharing `package`/`version`:

```
packages\certo\1.8.0\
    certo.exe
    certo-fmt.exe
```

```toml
# shims\certo.toml
package = "certo"
command = "certo.exe"
version = "1.8.0"
```

```toml
# shims\certo-fmt.toml
package = "certo"
command = "certo-fmt.exe"
version = "1.8.0"
```

Copy the shim binary under both names in `bin\` as usual. When you
switch versions, update **every** descriptor sharing that `package`
value together (a real `ion use` would automate this by scanning
`shims\*.toml` for matches) — don't leave `certo` on one version and
`certo-fmt` on another.

## Error reference

| Message | Meaning | Fix |
|---|---|---|
| `no argv[0]` | The OS didn't provide an invocation name at all — shouldn't happen in practice. | Investigate how the process was spawned. |
| `LOCALAPPDATA is not set` | The `LOCALAPPDATA` environment variable is missing and `ION_HOME` wasn't set either. | Set `ION_HOME` explicitly, or fix your environment. |
| `no shim descriptor for '<name>'` | Nothing at `shims\<name>.toml` exists, and no `ion.toml` pins that name either. | The tool was never shimmed — do step 5. |
| `'<name>' is pinned to <version> in ion.toml, but has never been shimmed globally ... run 'ion shim add <name> <package>@<version>' first` | An `ion.toml` pins a tool with no matching global descriptor. | Do step 5 for that tool name first; the pin alone isn't enough. |
| `malformed shim descriptor <path> (need package, command, version)` | The descriptor is missing one of the three required fields. | Check the file against the format in step 5b. |
| `<package>@<version> is not installed (looked for <path>)` | The resolved version (from the descriptor or a pin) has no matching folder under `packages\`. | Install that version (step 4), fix the typo in the descriptor/pin, or point a version alias there. |

Every error exits with code `127` and prints to stderr, prefixed
`ion-shim: `.

## Testing without touching your real `%LOCALAPPDATA%\Ion`

Set `ION_HOME` to any directory to redirect everything (`bin` isn't
part of this — only `shims\` and `packages\` are read relative to it).
Useful for trying out a layout before committing to it:

```powershell
$env:ION_HOME = "C:\scratch\fake_ion_home"
```
