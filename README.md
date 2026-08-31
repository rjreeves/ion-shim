# ion-shim

A single generic shim binary, in Certo, for Ion's version-manager `PATH`
trick: one tiny executable, copied under many names, that resolves the
active version of whatever it was invoked as and re-execs it — with real
stdin/stdout/stderr passthrough and exact exit-code propagation.

## Layout it expects

```
%LOCALAPPDATA%\Ion\
    bin\
        certo.exe        ← copy of ion_shim.exe, named "certo"
        flux.exe         ← copy of ion_shim.exe, named "flux"
    shims\
        certo.toml       ← which version "certo" currently resolves to
        flux.toml
    packages\
        certo\
            1.7.0\certo.exe
            1.8.0\certo.exe
```

Only `%LOCALAPPDATA%\Ion\bin` needs to be on `PATH`. Adding a new managed
tool is `ion shim add <name> <package>@<version>` (writes a `.toml` +
copies the shim binary under `<name>.exe`) — no `PATH` change, ever.
Switching versions (`ion use certo@1.7`) is a single-line edit to
`shims\certo.toml`; the shim binary itself never changes.

## Descriptor format (`shims\<name>.toml`)

Flat `key = "value"` lines only — no nesting, no arrays. `#` starts a
comment; blank lines are ignored.

```toml
package = "certo"
command = "certo.exe"
version = "1.8.0"
```

`ION_HOME` overrides `%LOCALAPPDATA%\Ion` (mainly for testing).

## Project-level pinning (`ion.toml`)

A project can pin a different version than the global `ion use` by
putting an `ion.toml` in its root:

```toml
# project-a/ion.toml
certo = "1.7.0"
```

```toml
# project-b/ion.toml
certo = "1.8.0"
```

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
