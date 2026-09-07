# ion-shim

A single generic shim binary, in Certo, for Ion's version-manager `PATH`
trick: one tiny executable, copied under many names, that resolves the
active version of whatever it was invoked as and re-execs it — with real
stdin/stdout/stderr passthrough and exact exit-code propagation.

This README covers the design; for step-by-step setup see
[docs/USAGE.md](docs/USAGE.md), and for a line-by-line explanation of
`src/ion_shim.cto` see [docs/CODE-WALKTHROUGH.md](docs/CODE-WALKTHROUGH.md).

## Layout it expects

```
%LOCALAPPDATA%\Ion\
    shim\
        ion_shim.exe     ← the master build artifact — never on PATH itself,
                             never invoked directly; every entry in bin\ below
                             is a copy of this one file, renamed
    bin\
        certo.exe        ← copy of shim\ion_shim.exe, named "certo"
        flux.exe         ← copy of shim\ion_shim.exe, named "flux"
    shims\
        certo.toml       ← which version "certo" currently resolves to
        flux.toml
    packages\
        certo\
            1.7.0\certo.exe
            1.8.0\certo.exe
```

Only `%LOCALAPPDATA%\Ion\bin` needs to be on `PATH`. `shim\ion_shim.exe`
deliberately lives outside it — it's the one-time build output that
every per-tool copy is stamped from, not something meant to run under
its own name. Keeping it here (rather than wherever it happened to be
built) means adding a new tool never requires recompiling from Certo
source: `ion shim add <name> <package>@<version>` just copies
`shim\ion_shim.exe` to `bin\<name>.exe` and writes the descriptor — no
`PATH` change, ever. Switching versions (`ion use certo@1.7`) is a
single-line edit to `shims\certo.toml`; the shim binary itself never
changes.

## Descriptor format (`shims\<name>.toml`)

Flat `key = "value"` lines only — no nesting, no arrays. `#` starts a
comment; blank lines are ignored.

```toml
package = "certo"
command = "certo.exe"
version = "1.8.0"
```

`ION_HOME` overrides `%LOCALAPPDATA%\Ion` (mainly for testing).

## Toolchains (a release with more than one binary)

Certo's own release ships several binaries — `certo`, `certo-fmt`,
`certo-lsp`, `xeq`, and more — and Rust's is bigger still (`rustc`,
`cargo`, `rustfmt`, `clippy`, `rust-analyzer`, ...). Fronting all of
them needs **no shim changes**: a toolchain is just several descriptors
that share `package` and `version` but each name their own `command`,
all reading from one shared version directory:

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

Verified: `certo.exe` and `certo-fmt.exe`, shimmed independently like
this, each resolve to their own binary inside the same
`packages\certo\1.8.0\` folder. `package`, `command`, and `version`
were already three independent fields per descriptor — nothing stopped
several descriptors from agreeing on two of them while differing on the
third. This mirrors how `rustup` treats a toolchain as one coordinated
bundle rather than versioning `rustc` and `cargo` separately.

That leaves exactly one real problem, and it belongs to whatever
installs packages, not the shim:

- **Coherent switching.** `ion use certo@1.9.0` needs to move every
  descriptor with `package = "certo"` to `1.9.0` together, so `certo`
  and `certo-fmt` never end up on different versions. This needs no
  extra bookkeeping beyond what already exists — since every descriptor
  already records its own `package`, switching just means scanning
  `shims\*.toml` for matches and rewriting all of them.
- **First install still needs a manifest.** Before any descriptors
  exist, something has to know that `certo@1.8.0` exposes `certo`,
  `certo-fmt`, `certo-lsp`, `xeq`, etc., so it knows which names to shim
  and which binaries to place in that shared version folder — that has
  to come from a release manifest (or, for Certo specifically, its own
  workspace member list) rather than being inferred.

## Version aliases (`latest`, `lts`, ...)

Both `version` in a shim descriptor and a pin in `ion.toml` can be a
named alias instead of an exact version, with **no shim code involved
at all** — a version string is just an opaque path segment to the
shim, so `version = "latest"` only works because `packages\certo\latest`
is a real directory. Concretely, that means a directory junction:

```
packages\certo\
    1.9.0\certo.exe
    latest\          ← junction (`New-Item -ItemType Junction`, no admin
                        rights needed, unlike a symlink) pointing at 1.9.0
```

Once that junction exists, `version = "latest"` in `shims\certo.toml`
and `certo = "latest"` in `ion.toml` both resolve correctly — verified
against a real junction, both from the global descriptor and from a
project pin. `lts` works identically; it's just another junction name.

This is deliberately pushed entirely onto whatever installs packages,
not the shim, for the same reason `package`/`command` identity lives in
the global descriptor rather than being inferred: the shim only
resolves, it never decides. It also sidesteps a real problem the shim
has no way to solve on its own — there's no way to tell, from a
directory name alone, which installed version was ever considered an
"LTS" release; only the installer has that knowledge. Whether the alias
is *live* (the junction gets re-pointed as new versions are installed,
so it silently tracks forward) or *frozen* (resolved once to a concrete
version and never updated) is entirely up to how the installer manages
it — the shim can't tell the difference and doesn't need to.

One caveat worth calling out even though nothing here enforces it: a
*live* floating alias inside a checked-in `ion.toml` undermines the
reproducibility pinning exists for in the first place — two people
building the same project at different times could silently get
different versions. Fine for a global default; worth avoiding in a
project pin meant to be reproducible.

## Project-level pinning (`ion.toml`)

A project can pin different versions than the global `ion use` by
putting a single `ion.toml` in its root — one file per project, listing
every tool that project cares about:

```toml
# project-a/ion.toml
certo = "1.7.0"
flux  = "1.2.0"
```

```toml
# project-b/ion.toml
certo = "1.8.0"
```

Each shim only reads its own line: `certo.exe` looks for a `certo =`
line and ignores `flux =`, and vice versa, so any number of tools can
share one file without stepping on each other. A `[tools]` header is
allowed for readability if you want one — the parser has no section
support, so it's silently skipped as a line with no `=` in it, same as
any other unrecognized line.

Only exact versions are supported (`certo = "1.7.0"`, not `^1.7` or
`>=1.7.0`) — a shim should resolve deterministically, not run a semver
solver, so there's no range/constraint syntax.

The shim walks up from the current directory looking for the nearest
`ion.toml`. The first one found is the project boundary:

- if it has a line for the invoked tool name, that version wins;
- if it exists but doesn't mention that tool, the shim stops climbing
  there anyway (that's still the project root) and falls back to the
  global `ion use` version — it will not skip past it to check a
  grandparent directory's `ion.toml`;
- if none is found before the filesystem root, the global version applies.

Only the version is pinned this way — `package` and `command` still come
from the global shim descriptor (`shims\<name>.toml`), since that's what
Ion wrote when the tool was first shimmed.

### Nested projects (monorepos)

A nested package with its own `ion.toml` overrides an ancestor's pin for
the same tool, since it's simply the nearer file found while climbing:

```toml
# monorepo/ion.toml
certo = "1.8.0"
```

```toml
# monorepo/packages/backend/ion.toml
certo = "1.7.0"
```

Invoking `certo` from `monorepo/packages/backend` resolves to `1.7.0`;
from `monorepo` itself (or any other sibling package without its own
pin), it resolves to `1.8.0`. This is "nearest file wins," not
"innermost value across all files wins": each `ion.toml` is a complete,
self-contained boundary. A nested file that pins `flux` but says
nothing about `certo` does **not** cause the climb to continue past it
to the root's `certo` pin — that case falls straight to the global
default instead (see the boundary rule above). This deliberately
matches `asdf`'s `.tool-versions` behavior rather than a cascading
workspace-inheritance model, so a pin never reaches further up the tree
than reading one file would suggest.

### When a pin and the global descriptor conflict

There's only one axis of override — version — and only one direction:
a project can narrow the global default, never redefine what a tool
name means. Concretely:

- The global descriptor (`shims\<name>.toml`) is mandatory and defines
  identity (`package`, `command`). `ion.toml` can never substitute for
  it or change what a name resolves to.
- If the global descriptor exists, `ion.toml`'s pinned version (if any)
  wins over the descriptor's `version`; otherwise the descriptor's
  version applies.
- If the global descriptor is **missing** but `ion.toml` pins that tool
  anyway, the shim doesn't silently ignore the pin — it fails with a
  specific message naming the orphaned pin (`'flux' is pinned to 1.2.0
  in ion.toml, but has never been shimmed globally ... run 'ion shim
  add flux <package>@1.2.0' first`) rather than the generic "no shim
  descriptor" error, since that would give no hint the pin exists at all.
- Whichever version wins, "is it actually installed" is checked the
  same way regardless of where the version came from.

## Build

```
certo src/ion_shim.cto -o ion_shim.exe
```

Copy (or hardlink) `ion_shim.exe` to `%LOCALAPPDATA%\Ion\bin\<name>.exe`
for each tool it should front. At runtime it reads its own `argv[0]`
to figure out which name it was invoked as.

## Stdlib gaps this surfaced

Building a *transparent* shim exposed several real gaps in Certo's
compiler, fixed or worked around here:

1. **`Stdlib.Process` had no inherited-stdio exec.** `Process.exec` only
   ever ran commands through `system()`/`popen()`, capturing stdout/stderr
   to temp files and returning them after the child exits — no live
   streaming, no real stdin. That's fatal for a shim (no progress bars,
   no interactive prompts, wrong stdout/stderr interleaving). Added
   `Process.execInherit(cmd, args): Int` to the compiler
   (`crates/stdlib/src/process.rs`, plus `seed.rs`/`effects_seed.rs`
   registration) — `CreateProcess` with inherited stdio handles on
   Windows, `fork`/`execvp` on POSIX — so the child is truly attached to
   the shim's own console. Verified with a real child process: live
   interleaved output, forwarded stdin, and exact exit code (7) round-tripped
   through the shim. This is upstream in the Certo checkout at
   `C:\Users\robert\Desktop\Certo` (uncommitted — not pushed since it
   wasn't asked for).

2. **`??` does not short-circuit.** `a ?? b` is documented as returning
   `a` unwrapped when `Some`, else `b` — but the compiler evaluates `b`
   unconditionally regardless of `a`. Confirmed with a minimal repro (an
   `[io]` side effect on the right-hand side runs even when the left is
   `Some(...)`). This breaks the common "unwrap-or-fail" idiom. Routed
   around it in `ion_shim.cto` via `match` (which *does* short-circuit
   correctly) instead of fixing the compiler, per instruction — worth
   fixing in `certo-codegen` separately since it likely affects other code
   relying on `??`.

3. **`Path.stem` doesn't strip the directory**, only the extension — so
   `Path.stem("C:\foo\bar.exe")` returns `C:\foo\bar` instead of `bar`.
   Worked around with `Path.stem(Path.basename(path))`; not fixed upstream.

4. **No way to get the current working directory.** Needed for project-level
   pinning (walking up from cwd looking for `ion.toml`) — there was no
   `cwd()`-equivalent in the stdlib at all. Added `getCurrentDir(): Text`
   to the compiler (`crates/stdlib/src/env.rs`, plus `seed.rs`
   registration) — `GetCurrentDirectoryA` on Windows, `getcwd` on POSIX.
   Left unmarked `[io]`, matching `getEnv`'s existing convention (reads
   process-local state that's static for the run, absent a `setCurrentDir`
   primitive, which doesn't exist either). Verified from two different
   working directories. Also uncommitted in the Certo checkout.
