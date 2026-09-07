# Using sym-shim

This is a practical, step-by-step guide to setting up and using the shim.
For the design rationale behind each of these behaviors, see the main
[README](../README.md); this document is about *doing*, not *why*.

**Scope note:** this repo ships two things — the shim binary
(`src/sym_shim.cto`) and a small `sym` CLI (`src/sym.cto`) implementing
`install`/`use`/`shim add`/`shim remove`/`shim list`/`pin`. `install`
can fetch a real GitHub release itself (with checksum verification) or
take an already-obtained local file — see step 4. Every section below
also shows the raw filesystem operations each `sym` command performs
— useful for understanding what's actually happening, or for scripting
around `sym` if you'd rather not depend on it.

## 1. Prerequisites

You need the `certo` compiler on `PATH` to build the shim. It's not
packaged anywhere conventional (no apt/choco/winget entry) — build it
from source, pinned to the exact commit `sym`/`sym_shim` are verified
against:

```bash
cargo install --git https://github.com/rjreeves/Certo --rev f0254c334efeae2404696d64da103ab306658433 certo
```

That commit is the first one with everything this project needs
(`Process.execInherit`, `getCurrentDir`, `HttpResponse.bodyBytes`) —
verified by building both `.cto` files against it directly. Certo has
exactly one tagged release (`v0.1.0`, June 2026), predating all three,
so a release build won't work here; pinning to a commit instead of
`--branch master` means a future, unrelated change to Certo's `master`
can't silently break this project's build. Re-pin to a newer commit
only after checking it still builds both `src/sym.cto` and
`src/sym_shim.cto` cleanly.

(Note: `certo` is a Cargo workspace with several binary-producing
crates, so the package name `certo` must be passed explicitly — `--bin
certo` alone is ambiguous and fails.)

## 2. Build the shim and the installer

```bash
certo src/sym_shim.cto -o sym_shim.exe
certo src/sym.cto -o sym.exe
```

`sym_shim.exe` is generic — it doesn't know which tool it's fronting
until you copy it under a specific name (step 5). `sym.exe` is the CLI
that does that copying and the rest of the bookkeeping for you; keep it
wherever's convenient on your own `PATH` (it isn't part of the `Sym\`
layout itself, unlike the shim).

## 3. Set up the directory layout

The shim expects this structure under `%LOCALAPPDATA%\Sym` (or wherever
`SYM_HOME` points, if you set that environment variable):

```
%LOCALAPPDATA%\Sym\
    shim\       <- the master sym_shim.exe build lives here, never on PATH
    bin\        <- per-tool copies of shim\sym_shim.exe; the only PATH entry needed
    shims\      <- one <name>.toml descriptor per shimmed tool
    packages\   <- actual installed binaries, one folder per version
```

Create the four folders once:

```powershell
New-Item -ItemType Directory -Force "$env:LOCALAPPDATA\Sym\shim"
New-Item -ItemType Directory -Force "$env:LOCALAPPDATA\Sym\bin"
New-Item -ItemType Directory -Force "$env:LOCALAPPDATA\Sym\shims"
New-Item -ItemType Directory -Force "$env:LOCALAPPDATA\Sym\packages"
```

Copy the binary you just built into its permanent home:

```powershell
Copy-Item sym_shim.exe "$env:LOCALAPPDATA\Sym\shim\sym_shim.exe"
```

Keeping one master copy here — separate from `bin\`, and not wherever
it happened to be built — means shimming a new tool later is always a
copy from `shim\sym_shim.exe`, never a recompile from Certo source.

Add `%LOCALAPPDATA%\Sym\bin` to your `PATH` — this is the **only** PATH
change ever required, no matter how many tools you shim later.
`shim\` is deliberately never added to `PATH`.

## 4. Install a real version of a tool

Two ways: give `sym install` a file you already have, or let it fetch
one.

**With a local file:**

```bash
sym install certo 1.8.0 C:\path\to\certo-1.8.0.exe certo.exe
```

This copies (binary-safe — see the code walkthrough for why that
matters) the given file to `packages\certo\1.8.0\certo.exe`, creating
the version folder if needed. The trailing `certo.exe` names the file
inside that folder; if omitted, it defaults to the source file's own
name. Equivalent by hand:

```powershell
New-Item -ItemType Directory -Force "$env:LOCALAPPDATA\Sym\packages\certo\1.8.0"
Copy-Item "C:\path\to\certo-1.8.0.exe" "$env:LOCALAPPDATA\Sym\packages\certo\1.8.0\certo.exe"
```

**Fetching instead** — omit the file argument, and write a source
config once per package first:

```toml
# %LOCALAPPDATA%\Sym\sources\certo.toml
provider = "github-release"
repo = "rjreeves/Certo"
tag = "v{version}"
asset = "certo-windows-x86_64.zip"
archive = "tar"
binary_path = "certo.exe"
```

```bash
sym install certo 1.8.0
```

This downloads `https://github.com/rjreeves/Certo/releases/download/v1.8.0/certo-windows-x86_64.zip`,
extracts it (via the `tar` binary bundled with Windows — no extra
install needed; `archive = "tar"` handles `.zip`, `.tar`, and
`.tar.gz`/`.tgz` alike, not just zip despite the name), and places
`certo.exe` the same way the local-file path would. For anything not
on GitHub — GitLab, a project's own server — use `provider = "url"`
with a fully-specified `url = "..."` template instead of
`repo`/`tag`/`asset`; see the README's "Fetching a package" section.
Add `%LOCALAPPDATA%\Sym\sources\certo\checksums.toml` with
a `<version> = "<sha256-hex>"` line to have the download verified
before it's installed — without one, `sym install` still works, just
prints an "unverified" warning. See the README's "Fetching a package"
section for the full format and the reasoning behind pinning checksums
separately rather than trusting whatever the release itself publishes.

## 5. Shim the tool

```bash
sym shim add certo certo 1.8.0
```

This copies `shim\sym_shim.exe` to `bin\certo.exe` (always re-copying,
even if already present — cheap, and picks up a rebuilt shim
automatically) and writes `shims\certo.toml`. An optional fifth
argument names the command inside the version folder if it isn't
`<name>.exe` (e.g. `sym shim add certo certo 1.8.0 certo-cli.exe`).
Equivalent by hand:

```powershell
Copy-Item "$env:LOCALAPPDATA\Sym\shim\sym_shim.exe" "$env:LOCALAPPDATA\Sym\bin\certo.exe"
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
to the real `1.8.0` binary. `sym shim list` prints every currently
shimmed name; `sym shim remove certo` deletes both `bin\certo.exe` and
`shims\certo.toml` (leaving installed packages alone, since another
name might still reference the same package/version).

## 6. Switch versions

Install the new version the same way as step 4, then:

```bash
sym use certo 1.9.0
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
make it yourself or `sym use` makes it for you.

## 7. Pin a version per-project

```bash
cd my-project
sym pin certo 1.7.0
```

Writes (or updates, preserving every other line including comments) a
`sym.toml` in the current directory to override the global version for
just that directory tree. Pass a directory as a third argument to pin
somewhere other than the current one: `sym pin certo 1.7.0 C:\path\to\project`.
By hand, that's just:

```toml
# my-project/sym.toml
certo = "1.7.0"
```

Any number of tools can share one file:

```toml
certo = "1.7.0"
flux  = "1.2.0"
```

Running `certo` from inside `my-project` (or any subdirectory under
it) now uses `1.7.0` regardless of the global setting — the shim walks
up from the current directory looking for the nearest `sym.toml`. See
the README's "Project-level pinning" and "Nested projects" sections for
the exact boundary rules (in short: the nearest file found wins
entirely, it doesn't merge with parent files).

**The tool must already be shimmed globally before you can pin it.**
`sym.toml` only overrides which *version* runs — it can't invent a
`package`/`command` mapping that doesn't exist yet in `shims\`. If you
pin a tool that was never shimmed, you'll get a specific error telling
you so (see the error reference below).

## 8. Set up a version alias (`latest`, `lts`, ...)

Create a directory junction (not a symlink — junctions don't need admin
rights):

```powershell
New-Item -ItemType Junction `
  -Path "$env:LOCALAPPDATA\Sym\packages\certo\latest" `
  -Target "$env:LOCALAPPDATA\Sym\packages\certo\1.9.0"
```

Now `version = "latest"` works in either a descriptor or a `sym.toml`
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
value together (a real `sym use` would automate this by scanning
`shims\*.toml` for matches) — don't leave `certo` on one version and
`certo-fmt` on another.

## Error reference

| Message | Meaning | Fix |
|---|---|---|
| `no argv[0]` | The OS didn't provide an invocation name at all — shouldn't happen in practice. | Investigate how the process was spawned. |
| `LOCALAPPDATA is not set` | The `LOCALAPPDATA` environment variable is missing and `SYM_HOME` wasn't set either. | Set `SYM_HOME` explicitly, or fix your environment. |
| `no shim descriptor for '<name>'` | Nothing at `shims\<name>.toml` exists, and no `sym.toml` pins that name either. | The tool was never shimmed — do step 5. |
| `'<name>' is pinned to <version> in sym.toml, but has never been shimmed globally ... run 'sym shim add <name> <package> <version>' first` | A `sym.toml` pins a tool with no matching global descriptor. | Do step 5 for that tool name first; the pin alone isn't enough. |
| `malformed shim descriptor <path> (need package, command, version)` | The descriptor is missing one of the three required fields. | Check the file against the format in step 5b. |
| `<package>@<version> is not installed (looked for <path>)` | The resolved version (from the descriptor or a pin) has no matching folder under `packages\`. | Install that version (step 4), fix the typo in the descriptor/pin, or point a version alias there. |

Every error exits with code `127` and prints to stderr, prefixed
`sym-shim: `.

## Testing without touching your real `%LOCALAPPDATA%\Sym`

Set `SYM_HOME` to any directory to redirect everything (`bin` isn't
part of this — only `shims\` and `packages\` are read relative to it).
Useful for trying out a layout before committing to it:

```powershell
$env:SYM_HOME = "C:\scratch\fake_sym_home"
```
