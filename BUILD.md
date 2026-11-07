# Building `omp` from source

Manual, from-scratch build of the `omp` coding-agent CLI. The end product is a
single standalone executable: `packages/coding-agent/dist/omp`.

Repository: `noctilust/omp-nebius-provider` (fork, default branch
`feat/nebius-token-factory`) tracking upstream `can1357/oh-my-pi`.

The build is two stages, in this order:

1. **Native addon** — `packages/natives` compiles `crates/pi-natives` as a Rust
   N-API `.node` addon for the host platform
   (`packages/natives/scripts/build-bindings.ts`).
2. **CLI binary** — `packages/coding-agent` bundles the TypeScript CLI with
   `bun build --compile` and embeds the stage-1 addon, the docs index, and the
   export-viewer assets (`packages/coding-agent/scripts/build-binary.ts`), then
   ad-hoc code-signs the result on macOS with
   `scripts/macos-entitlements.plist`.

This mirrors the CI sequence in `scripts/install-tests/binary.dockerfile`.

## Prerequisites

| Tool | Needed for | Install | Check |
| --- | --- | --- | --- |
| git | cloning | macOS: `xcode-select --install`; Linux: `apt-get install git` | `git --version` |
| Bun ≥ 1.4.4 | workspace install, TS bundling, compiled-binary toolchain (`packageManager` field) | `brew install bun` or `curl -fsSL https://bun.sh/install \| bash` | `bun --version` |
| Rust via rustup | compiling `crates/pi-natives` | `curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs \| sh`, keep `~/.cargo/bin` on `PATH` | `rustup show active-toolchain` (run inside the repo) |
| Xcode Command Line Tools (macOS) | clang/linker, `codesign` for ad-hoc signing, Swift for the Apple Foundation Models bridge | `xcode-select --install` | `xcode-select -p` |
| CMake + Ninja | native C/C++ dependencies (`CMAKE_GENERATOR=Ninja` is pinned in `.cargo/config.toml`; Xcode's GNU make 3.81 can deadlock on cargo's jobserver pipe) | macOS: `brew install cmake ninja`; Linux: `apt-get install cmake ninja-build`; Windows: both ship inside VS Build Tools | `cmake --version`, `ninja --version` |
| Linux build toolchain | C compiler/linker for native deps | `apt-get install -y curl ca-certificates unzip build-essential` | `cc --version` |

The Rust toolchain is pinned by `rust-toolchain.toml` (`nightly-2026-10-06`
plus `rustfmt`, `clippy`, `rust-analyzer`); rustup downloads it automatically
the first time `cargo` runs inside the repo, so the first build needs network
access. The cross targets listed there are only for release/cross builds — the
host build uses the host target.

Windows hosts build the host addon through the same Cargo/N-API path; the
script locates `cmake`/`ninja` inside VS Build Tools itself.

## Build steps

```bash
# 1. Clone
git clone https://github.com/noctilust/omp-nebius-provider.git
cd omp-nebius-provider

# Optional: keep upstream reachable for syncing
git remote add upstream https://github.com/can1357/oh-my-pi.git
git fetch upstream --prune --tags

# 2. Workspace dependencies (add --frozen-lockfile for CI-exact versions)
bun install

# 3. Host native addon → packages/natives/native/pi_natives.<os>-<arch>[-modern|-baseline].node
bun --cwd=packages/natives run build      # same as: bun run build:native (from the root)

# 4. Standalone binary → packages/coding-agent/dist/omp
bun --cwd=packages/coding-agent run build # same as: bun run build (inside that package)

# 5. Install on PATH (~/.local/bin is what the official installer uses)
mkdir -p ~/.local/bin
cp packages/coding-agent/dist/omp ~/.local/bin/omp.new
mv ~/.local/bin/omp.new ~/.local/bin/omp   # rename keeps a running omp on its old inode
```

The first stage-3 build compiles the whole Rust dependency graph and takes
minutes; later rebuilds reuse the workspace `target/` directory. Re-run
stage 3 after changing anything under `crates/` or `packages/natives/`, and
stage 4 after changing TypeScript.

For a dev loop that runs the CLI straight from source (no standalone binary),
the README's "Getting started from source" section covers `bun setup` (install
→ native addon → link the dev wrapper `omp` into Bun's global bin) and
`bun dev`.

Note: the stage-3 host build also regenerates the committed declarations
(`packages/natives/native/index.js` / `index.d.ts`) from the Rust sources. If
the committed declarations lag the Rust doc comments, `index.d.ts` shows as
modified after a build (types-only — no runtime effect). The built `.node`
itself is git-ignored.

## Verify

```bash
packages/coding-agent/dist/omp --version
packages/coding-agent/dist/omp --smoke-test   # pings every bundled worker (stats sync, tiny title, TTS/STT, JS eval, LSP mux, …) and prints "smoke-test: ok"

# After installing, and without touching your real ~/.omp state:
HOME=$(mktemp -d) XDG_DATA_HOME=$(mktemp -d) omp --version
```

`--smoke-test` is the same probe wired into `ci:test:smoke` and the install
tests, so it also catches worker-host packaging regressions.

## Outputs

| Artifact | Path | Notes |
| --- | --- | --- |
| Native addon | `packages/natives/native/pi_natives.<platform>-<arch>[-modern\|-baseline].node` | x64 hosts pick the `modern`/`baseline` ISA variant via AVX2 detection; non-x64 has no suffix. Stamped with the version from `packages/natives/package.json`. |
| Standalone CLI | `packages/coding-agent/dist/omp` | Embeds the JS bundle (precompiled bytecode), the native addon, the docs index, and export-viewer assets. Ad-hoc signed on macOS. |

## Build options (environment variables)

- `OMP_NATIVE_CARGO_PROFILE` — Cargo profile for the host addon (default
  `local`; `ci` is stripped; profiles are defined in the root `Cargo.toml`).
- `OMP_NATIVE_BUILD_BACKEND=bazel` — opt into the hermetic Bazel addon build
  (needs `bazelisk`); the default for `host` is the local Cargo/N-API path, and
  `OMP_NATIVE_BUILD_BACKEND=cargo` forces it.
- `OMP_NATIVE_FEATURES` — extra Cargo features passed to `napi build`, e.g.
  `wayland-pipewire` on Linux.
- `CROSS_TARGET=darwin-x64|darwin-arm64|linux-x64|linux-arm64|win32-x64|win32-arm64`
  — cross-compile the CLI binary (`bun --cwd=packages/coding-agent run build`);
  requires that platform's addon artifact, so it is a release-pipeline path,
  not a plain host build.
- `BUN_NO_CODESIGN_MACHO_BINARY=1` — skip *all* signing on macOS (Bun's
  built-in signing and the explicit entitlements re-sign), leaving the binary
  unsigned.

## Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| `Failed to load pi_natives native addon for <platform>-<arch>` at startup | addon missing or built for another platform | run stage 3; confirm the `.node` exists in `packages/natives/native/` |
| `cargo metadata failed to run` / `cargo: command not found` | `~/.cargo/bin` not on `PATH` | `export PATH="$HOME/.cargo/bin:$PATH"` |
| Native build fails inside CMake/Ninja | generators missing | install `cmake` and `ninja` |
| `cargo:warning=no Swift 6.4+ toolchain with the macOS 27 SDK; Apple Foundation Models support is stubbed out` | older Xcode/CLT | benign unless you need Apple Foundation Models; install current Xcode CLT to enable the bridge |
| `bun install --frozen-lockfile` fails | local dependency edits changed `bun.lock` | run plain `bun install` (or restore `bun.lock`) |
| CLI refuses to start, requires a newer Bun | Bun too old | upgrade Bun (`bun upgrade`); the floor is 1.4.4 |
| Pinned nightly toolchain fails to download | offline / rustup missing the pin | `rustup toolchain install nightly-2026-10-06 --component rustfmt --component clippy --component rust-analyzer` |

See `docs/natives-build-release-debugging.md` for the full addon
build/load/embedding internals and Bazel-release details.
