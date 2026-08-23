# micropython-native-ci

Shared GitHub Actions building blocks for building and testing MicroPython
**natmod** (dynamically loadable native `.mpy` modules, built against
`py/dynruntime.mk`) and **usermod** (`USER_C_MODULES`, compiled straight
into a port's own firmware build) C extensions.

## Why this repo exists

MicroPython itself already defines two standard, unrelated build
mechanisms for a native C extension:

- **natmod** -- `natmod/Makefile` includes MicroPython's own
  `py/dynruntime.mk` and is parameterised by `ARCH=`. It produces a
  runtime-loadable `.mpy` per target architecture. See MicroPython's own
  `examples/natmod/`.
- **usermod** -- `usermod/micropython.cmake` + `usermod/micropython.mk`,
  pointed at via `USER_C_MODULES=` on a port's own build. Compiled into the
  firmware image itself. See MicroPython's
  [`docs/develop/cmodules.rst`](https://docs.micropython.org/en/latest/develop/cmodules.html).

[`ballistics-lab/micropython-bclibc`](https://github.com/ballistics-lab/micropython-bclibc),
[`o-murphy/micropython-wasm3`](https://github.com/o-murphy/micropython-wasm3)
and [`o-murphy/a7p`](https://github.com/o-murphy/a7p) each already follow
that same `natmod/` + `usermod/` layout -- that part was never the problem.
What diverged was the CI *around* it: each repo's GitHub Actions workflow
was hand-copied into the next and then evolved independently, so the same
~10-architecture build matrix, the same toolchain-install steps and the
same real-ARM-Linux test trick ended up as three separate, slowly drifting
copies (different `actions/checkout` versions, different path filters, one
bug fixed in one repo and not the other two).

This repo is the shared home for the parts that are genuinely identical
across all three -- so a fix or an improvement lands once, and every
consuming repo picks it up deliberately by bumping the tag it's pinned to,
instead of by hand-patching three YAML files that have already started to
disagree with each other.

## What's here

### Composite actions (`.github/actions/`)

Every table below is the action's complete input surface -- if it isn't
listed here, the action doesn't accept it. `MPY_DIR` in a "Requires"
line means `fetch-micropython` or `clone-micropython` (this repo's own)
must have already run in the same job; a composite action step can't set
an env var that steps *before* it will see, only ones after.

#### `fetch-micropython`

Downloads and extracts a MicroPython release tarball, exports `MPY_DIR`.
Use for a plain natmod build or a unix-port build; the tarball already
vendors every port's `lib/` submodules, so no submodule init is needed.
Not usable on a Windows runner outside MSYS2 -- it shells out to `wget`,
which plain Git Bash doesn't have (see `build-usermod-windows-arch` below
for why the Windows actions never call it either).

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `mpy_tag` | yes | -- | MicroPython release tag, e.g. `v1.28.0` |

No outputs; exports `MPY_DIR` to `$GITHUB_ENV` as a side effect.

#### `clone-micropython`

Shallow git-clones a MicroPython release branch instead of fetching a
tarball, with a chosen set of submodules initialised, and exports
`MPY_DIR`. Use this when the build needs a submodule the release tarball
doesn't vendor (`lib/pico-sdk` for an rp2 firmware build, for instance) --
or, as a7p and now bclibc/wasm3's own webassembly/rp2040/windows-adjacent
jobs use it, any time the caller needs `MPY_DIR` set without dragging in
`fetch-micropython`'s `wget` dependency.

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `mpy_tag` | yes | -- | MicroPython release tag |
| `submodules` | no | `''` | Space-separated submodules to `git submodule update --init` (empty = skip) |
| `pico_sdk_submodules` | no | `'false'` | Also run `git -C lib/pico-sdk submodule update --init` (rp2040 builds) |
| `path` | no | `'micropython'` | Clone destination, relative to the workspace root -- override when the caller's own repo already has a top-level directory of that name (a7p passes `path: mpy`, since its own MicroPython subtree already lives at `micropython/`) |

No outputs; exports `MPY_DIR` to `$GITHUB_ENV` as a side effect.

#### `build-natmod-arch`

Installs whatever toolchain a single `dynruntime.mk` `ARCH` needs (plain
apt package, the `xtensa-lx106` tarball, or esp-idf -- dispatched per
arch, matching `dynruntime.mk`'s own `CROSS` choices), builds `mpy-cross`,
then runs `make ARCH=<arch> dist` in the natmod directory.

Requires: `MPY_DIR` (see above) and the calling repo already checked out,
submodules included if the natmod Makefile needs any.

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `arch` | yes | -- | `x64`, `x86`, `armv6m`, `armv7m`, `armv7emsp`, `armv7emdp`, `rv32imc`, `rv64imc`, `xtensa`, or `xtensawin` (no `aarch64` -- `dynruntime.mk` has none as of MicroPython ≤ v1.28; build that via a usermod instead) |
| `natmod_dir` | no | `natmod` | Path to the directory containing `natmod/Makefile`, relative to the workspace root (a7p passes `micropython/natmod`) |
| `esp_idf_ver` | no | `v5.4` | esp-idf tag to install for the `xtensawin` toolchain |
| `extra_pip` | no | `''` | Extra space-separated pip packages, alongside `pyelftools`/`ar` (always installed -- `mpy_ld.py` needs them for every ARCH) |
| `pre_build_command` | no | `''` | Shell command run once inside `natmod_dir`, after `mpy-cross` and before `make dist` (a7p uses `make fetch-nanopb`) |

No outputs.

#### `build-usermod-unix-arch`

The unix-port cross-compile matrix for a `USER_C_MODULES` usermod: `x64`,
`x86` (32-bit), `aarch64`, `armhf`, or `mipsel`. Installs the arch's
toolchain (apt package, qemu-user-static for the emulated ones, a
from-source libffi for the statically-linked ones), builds `mpy-cross`,
then runs the port build.

Requires: `MPY_DIR` and checkout, same as `build-natmod-arch`. The
caller's own matrix still has to choose `runs-on:` per arch
(`ubuntu-24.04-arm` for `aarch64`/`armhf` -- both execute natively there,
not under an emulator; `ubuntu-latest` for the rest, `mipsel` included --
it stays under `qemu-user-static`, since GitHub has no mips runner) -- a
composite action can't pick its own runner.

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `arch` | yes | -- | `x64`, `x86`, `aarch64`, `armhf`, or `mipsel` |
| `user_c_modules` | no | `''` → `$GITHUB_WORKSPACE` | Value for `USER_C_MODULES=` |
| `frozen_manifest` | no | `''` → `$GITHUB_WORKSPACE/usermod/manifest.py` | Value for `FROZEN_MANIFEST=` |
| `extra_make_args` | no | `''` | Extra space-separated `VAR=value` pairs appended to the build command (e.g. bclibc's `MP_BCLIBC_PRECISION=double`) |
| `build_dir` | no | `''` → `$GITHUB_WORKSPACE/usermod/build/<arch>` | Value for `BUILD=`. A bare relative value (no leading `/`, e.g. `build-wasm3`) resolves against `$MPY_DIR/ports/unix` instead, the same way a bare `BUILD=` on the command line always did |
| `variant` | no | `standard` | Value for `VARIANT=`. A caller building against upstream's own `VARIANT=coverage` recipe (a7p's armhf/mipsel qemu legs used to) overrides this |

| Output | Description |
| --- | --- |
| `build_dir` | The `BUILD=` directory actually used (resolved default included), so the caller can find the built binary without recomputing it |

#### `build-usermod-windows-arch`

The `ports/windows` half of the same usermod build, run inside an MSYS2
shell (every step is `shell: msys2 {0}`): builds `mpy-cross` then the
port itself, including the four CLANGARM64-only overrides every
consuming repo's Windows row needed (`LDFLAGS_ARCH`/`COMPILER_TARGET`
because CLANGARM64 links via clang+lld rather than GNU ld/gcc,
`STRIP=""`/`SIZE="true"` because that toolchain ships neither binary).

Deliberately narrower than `build-usermod-unix-arch`: fetching MicroPython
and setting up MSYS2 both stay the caller's own job. Requires:

- `MPY_DIR`, exported to a **POSIX-style path** (no backslashes -- MSYS2
  bash's own escape character eats them on any unquoted command line built
  from a native `D:\a\...` value, a real failure documented in every
  caller this was extracted from). Neither `fetch-micropython` nor
  `clone-micropython` is safe to use for this on a Windows runner as-is:
  the former shells out to `wget`, which plain Git Bash doesn't have, and
  the latter's own `$GITHUB_WORKSPACE`-derived `MPY_DIR` is the native
  backslash form. Every caller this was extracted from sets `MPY_DIR`
  itself with an inline `curl`+`$(pwd)` step instead.
- `msys2/setup-msys2` already run in the calling job, with the target
  `msystem`. This action's own steps can't do it for themselves --
  they're composite-action steps, so their `shell:` is fixed at
  `msys2 {0}` regardless of what ran before them in the *calling* job,
  and that shell wrapper only exists on `PATH` once `setup-msys2` has
  put it there.

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `user_c_modules` | no | `$(pwd)` | Value for `USER_C_MODULES=` |
| `frozen_manifest` | no | `$(pwd)/usermod/manifest.py` | Value for `FROZEN_MANIFEST=` |
| `extra_make_args` | no | `''` | Extra space-separated `VAR=value` pairs, e.g. a custom `PROG=` (wasm3 uses `PROG=micropython-wasm3.exe`) |
| `build_dir` | no | `build-standard` | Value for `BUILD=` -- a bare relative value, resolving against `$MPY_DIR/ports/windows` |
| `cflags_extra` | no | `''` | Value for `CFLAGS_EXTRA=` on the main build only (not `mpy-cross`), e.g. `-Wno-error` for CLANGARM64 |
| `variant` | no | `''` | Value for `VARIANT=` on the main build only, omitted from the command line entirely when empty. None of the three current callers ever pass this -- `ports/windows` has no `variants/<name>/` split in any of them, unlike the unix port -- it's here for a future caller whose own fork of the port does define one |

Every path input defaults to a `$(pwd)`-relative value, never an absolute
one, for the same backslash reason `MPY_DIR` has to be POSIX-style.

| Output | Description |
| --- | --- |
| `build_dir` | The `BUILD=` directory actually used (the input, verbatim) |

### Usage example

```yaml
jobs:
  build:
    strategy:
      fail-fast: false
      matrix:
        arch: [x64, x86, armv6m, armv7m, armv7emsp, armv7emdp, rv32imc, rv64imc, xtensa, xtensawin]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
        with:
          submodules: recursive

      - uses: ballistics-lab/micropython-native-ci/.github/actions/fetch-micropython@v0.1.0
        with:
          mpy_tag: v1.28.0

      - uses: ballistics-lab/micropython-native-ci/.github/actions/build-natmod-arch@v0.1.0
        with:
          arch: ${{ matrix.arch }}
          # natmod_dir: natmod              # default; a7p passes micropython/natmod
          # pre_build_command: make fetch-nanopb   # a7p-only, runs before `make dist`

      - uses: actions/upload-artifact@v7
        with:
          name: my-module-${{ matrix.arch }}
          path: natmod/build/${{ matrix.arch }}*/
          if-no-files-found: error
```

That replaces roughly 70 lines of per-arch toolchain-install boilerplate
(apt package selection, the xtensa tarball fetch, the esp-idf install +
the esp-idf-venv-shadows-system-pip fix) with one `uses:` block per matrix
leg, identical across every consuming repo.

Artifact upload is left to the caller on purpose -- artifact names and the
exact glob under `natmod/build/` differ per repo/module and aren't part of
the shared contract.

## Conventions this repo assumes

A module following the natmod/usermod layout looks like:

```
natmod/
  Makefile              # includes py/dynruntime.mk, dispatches on ARCH=
usermod/
  micropython.cmake
  micropython.mk
  manifest.py
```

`build-natmod-arch` only assumes `natmod/Makefile` (or whatever
`natmod_dir` points at) accepts `ARCH=` and `MPY_DIR=` and has a `dist`
target that drops the built `.mpy` under `build/<arch>*/`. Nothing here
assumes a specific module name, precision scheme, or test framework --
those stay in the consuming repo.

## Roadmap

Not done yet, deliberately -- each of these needs to be verified against
real CI in the consuming repo, not just written and trusted:

- A reusable `workflow_call` for the full natmod build+test matrix
  (build-natmod-arch already covers the highest-drift part of it).
- A reusable `workflow_call` for the "armv7emsp / armv7emdp on real ARM
  Linux" test job -- currently near-identical hand-copied YAML in all
  three source repos.
- A small scaffolding script to generate a new `natmod/` + `usermod/`
  skeleton for a fresh module, wired to the actions above.

## Versioning

Pin consumers to a tag (`@v0.1.0`), not `@main` and not a commit SHA.
Bumping the tag a consumer references is a deliberate, visible edit in
that repo, same as bumping any other CI dependency -- a change here never
silently changes what three other repos' builds do. `v0.1.0` is the
current tag; each of bclibc, a7p and micropython-wasm3 pin to it.
