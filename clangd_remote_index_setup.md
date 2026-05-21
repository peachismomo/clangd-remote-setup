# Linux clangd remote index setup

This setup keeps a single OrcaSlicer index server running locally and lets worktrees use it through clangd's remote index support. It is intended to reduce background indexing CPU/RAM usage in each worktree while preserving normal clangd parsing for files you open.

The commands below are written for Ubuntu/Debian-style Linux systems. On other distributions, install the equivalent packages with that distribution's package manager.

## What this provides

- `clangd-indexer` builds a project-wide Dex index from `compile_commands.json`.
- `clangd-index-server` serves that index over localhost using gRPC.
- VS Code's clangd extension connects to that server instead of background-indexing every worktree.
- Each worktree gets its own `MountPoint`, so navigation opens files in the current worktree.

## Required system packages

Install the normal command-line tools used by this workflow:

```bash
sudo apt update
sudo apt install \
  build-essential \
  ca-certificates \
  cmake \
  curl \
  git \
  ninja-build \
  pkg-config \
  python3 \
  unzip \
  xz-utils
```

Optional, but useful when building LLVM locally:

```bash
sudo apt install lld
```

If building clangd with remote index support from source, also install gRPC/protobuf development packages:

```bash
sudo apt install \
  libgrpc++-dev \
  libprotobuf-dev \
  protobuf-compiler \
  protobuf-compiler-grpc
```

For OrcaSlicer itself, this guide assumes you already have:

```bash
build/compile_commands.json
deps/build/OrcaSlicer_dep/usr/local
```

If `compile_commands.json` is missing, configure OrcaSlicer with CMake export enabled:

```bash
cmake -S . -B build -G "Ninja Multi-Config" \
  -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
```

The full OrcaSlicer dependency build is separate from this clangd remote-index setup.

## Check whether you already have remote-enabled clangd

Remote index support requires a clangd build with gRPC support. Check:

```bash
clangd --version
```

Useful output contains something like:

```text
Features: linux+grpc
```

Also check that the indexing tools exist:

```bash
command -v clangd-indexer
command -v clangd-index-server
```

If the tools are missing, or `clangd --version` does not show `grpc`, use one of the installation options below.

Do not assume the distro `clangd` package is enough. Some LLVM/Debian-style packages ship normal clangd but not remote-index-enabled clangd or the separate `clangd-indexer` / `clangd-index-server` tools.

## Option A: use a prebuilt clangd release

The clangd release artifacts include remote index support and separate indexing tools. Download the latest Linux release from:

```text
https://github.com/clangd/clangd/releases
```

You need both:

- the clangd Linux archive
- `clangd_indexing_tools.zip`

Install them under a user-local prefix, for example:

```bash
mkdir -p "$HOME/opt/llvm-clangd-remote"
```

After unpacking/copying, verify:

```bash
"$HOME/opt/llvm-clangd-remote/bin/clangd" --version
"$HOME/opt/llvm-clangd-remote/bin/clangd-indexer" --version
"$HOME/opt/llvm-clangd-remote/bin/clangd-index-server" --version
```

Also verify the clang builtin headers are present:

```bash
find "$HOME/opt/llvm-clangd-remote/lib/clang" -path '*/include/float.h' -type f
```

If `float.h` is missing, the install is incomplete and indexing will fail on standard headers.

## Option B: build LLVM/clangd from source (More reliable, but longer)

Use this when your distro package does not include remote index support or the release artifact does not fit your environment.

Clone LLVM:

```bash
mkdir -p "$HOME/repos"
git clone https://github.com/llvm/llvm-project.git "$HOME/repos/llvm-project"
```

Configure a remote-enabled clangd build:

```bash
cmake -S "$HOME/repos/llvm-project/llvm" \
  -B "$HOME/repos/llvm-project/build" \
  -G Ninja \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_INSTALL_PREFIX="$HOME/opt/llvm-clangd-remote" \
  -DLLVM_ENABLE_PROJECTS="clang;clang-tools-extra" \
  -DLLVM_TARGETS_TO_BUILD="X86" \
  -DCLANGD_ENABLE_REMOTE=On \
  -DLLVM_ENABLE_ASSERTIONS=Off
```

If you installed `lld`, you can add:

```bash
-DLLVM_USE_LINKER=lld
```

On low-memory machines, reduce parallel linking:

```bash
-DLLVM_PARALLEL_LINK_JOBS=1
```

If CMake cannot find gRPC, pass its install location explicitly:

```bash
-DGRPC_INSTALL_PATH=/usr
```

Build and install:

```bash
cmake --build "$HOME/repos/llvm-project/build" \
  --target clangd clangd-indexer clangd-index-server \
  -- -j"$(nproc)"

cmake --build "$HOME/repos/llvm-project/build" \
  --target install \
  -- -j"$(nproc)"
```

If the install target does not place all three binaries in the prefix, copy them manually:

```bash
mkdir -p "$HOME/opt/llvm-clangd-remote/bin"
cp "$HOME/repos/llvm-project/build/bin/clangd" \
   "$HOME/repos/llvm-project/build/bin/clangd-indexer" \
   "$HOME/repos/llvm-project/build/bin/clangd-index-server" \
   "$HOME/opt/llvm-clangd-remote/bin/"
```

Make sure the matching clang resource directory is installed. This is required for builtin headers like `float.h`:

```bash
mkdir -p "$HOME/opt/llvm-clangd-remote/lib/clang"
cp -a "$HOME/repos/llvm-project/build/lib/clang/"* \
      "$HOME/opt/llvm-clangd-remote/lib/clang/"
```

Verify:

```bash
"$HOME/opt/llvm-clangd-remote/bin/clangd" --version
find "$HOME/opt/llvm-clangd-remote/lib/clang" -path '*/include/float.h' -type f
```

Expected clangd features include:

```text
Features: linux+grpc
```

## Installed layout

The helper scripts live outside the repo:

```bash
~/scripts/clangd-server/generate-remote-clangd-index.sh
~/scripts/clangd-server/run-clangd-index-server.sh
~/scripts/clangd-server/setup-worktree-clangd-remote-index.sh
```

The remote-index clangd toolchain lives here:

```bash
~/opt/llvm-clangd-remote/bin/clangd
~/opt/llvm-clangd-remote/bin/clangd-indexer
~/opt/llvm-clangd-remote/bin/clangd-index-server
~/opt/llvm-clangd-remote/lib/clang/23/include
```

The `lib/clang/23/include` resource directory is required. Without it, indexing fails with errors like:

```text
fatal error: 'float.h' file not found
```

On another LLVM version, replace `23` with the version printed by:

```bash
"$HOME/opt/llvm-clangd-remote/bin/clangd-indexer" --version
```

## Install the helper scripts

This setup keeps the helper scripts in a global user directory:

```bash
mkdir -p "$HOME/scripts/clangd-server"
```

If you have the packaged archive, install it like this:

```bash
mkdir -p /tmp/orca-clangd-server-scripts
unzip scripts.zip -d /tmp/orca-clangd-server-scripts
cd /tmp/orca-clangd-server-scripts
chmod +x cpy.sh
./cpy.sh
```

The archive layout is:

```text
scripts.zip
  generate-remote-clangd-index.sh
  run-clangd-index-server.sh
  setup-worktree-clangd-remote-index.sh
  cpy.sh
```

`cpy.sh` copies the helper scripts to `$HOME/scripts/clangd-server` and runs `chmod +x` on the installed copies.

If installing manually instead, copy these scripts there:

```bash
generate-remote-clangd-index.sh
run-clangd-index-server.sh
setup-worktree-clangd-remote-index.sh
```

Make them executable:

```bash
chmod +x "$HOME/scripts/clangd-server/"*.sh
```

## Shell functions

Add the following functions to `~/.bashrc`. Replace the repo paths if your checkouts live somewhere else.

```bash
export ORCA_CLANGD_SCRIPT_DIR="$HOME/scripts/clangd-server"

orca_clangd_root() {
  case "${1:-}" in
    -p|--priv) echo "/home/peach/repos/OrcaSlicer_priv" ;;
    *) echo "/home/peach/repos/OrcaSlicer" ;;
  esac
}

genindex() {
  local root
  root="$(orca_clangd_root "${1:-}")"
  (cd "$root" && "$ORCA_CLANGD_SCRIPT_DIR/generate-remote-clangd-index.sh")
}

clangdserver() {
  local root
  root="$(orca_clangd_root "${1:-}")"
  (cd "$root" && "$ORCA_CLANGD_SCRIPT_DIR/run-clangd-index-server.sh")
}

mountclangd() {
  "$ORCA_CLANGD_SCRIPT_DIR/setup-worktree-clangd-remote-index.sh"
}

cpvscode() {
  local root
  root="$(orca_clangd_root "${1:-}")"
  rm -rf .vscode
  cp -a "$root/.vscode" .
}

orcaln() {
  local root
  root="$(orca_clangd_root "${1:-}")"
  mkdir -p deps
  ln -sfnT "$root/deps/build" deps/build
}

setupworktree() {
  local mode="${1:-}"
  orcaln "$mode" && cpvscode "$mode" && mountclangd
}

startclangd() {
  local mode="${1:-}"
  genindex "$mode" && clangdserver "$mode"
}
```

Reload the shell after editing `~/.bashrc`:

```bash
source ~/.bashrc
```

## Public repo workflow

Generate the index from the main checkout:

```bash
genindex
```

This writes:

```bash
/home/peach/repos/OrcaSlicer/.clangd-remote-index/clangd.dex
```

Start the local index server in a separate terminal and keep it running:

```bash
clangdserver
```

The default server address is:

```text
127.0.0.1:5900
```

Set up a public OrcaSlicer worktree:

```bash
cd /path/to/public/worktree
setupworktree
```

That does three things:

```bash
mkdir -p deps
ln -sfnT /home/peach/repos/OrcaSlicer/deps/build deps/build
rm -rf .vscode && cp -a /home/peach/repos/OrcaSlicer/.vscode .
~/scripts/clangd-server/setup-worktree-clangd-remote-index.sh
```

The worktree `.clangd` will contain:

```yaml
CompileFlags:
  CompilationDatabase: build

Index:
  Background: Skip
  External:
    Server: 127.0.0.1:5900
    MountPoint: /path/to/current/worktree/
```

## Private repo workflow

Pass `-p` or `--priv` to use `/home/peach/repos/OrcaSlicer_priv`.

Generate the private index:

```bash
genindex -p
```

Start the private index server:

```bash
clangdserver -p
```

Set up a private worktree:

```bash
cd /path/to/private/worktree
setupworktree -p
```

## Running public and private servers at the same time

Both modes default to `127.0.0.1:5900`, so only one can run on the default port.

Use a different port for the second server:

```bash
CLANGD_INDEX_SERVER_ADDRESS=127.0.0.1:5901 clangdserver -p
```

Mount worktrees that should use that server with the same address:

```bash
cd /path/to/private/worktree
CLANGD_INDEX_SERVER_ADDRESS=127.0.0.1:5901 mountclangd
```

If you use `setupworktree -p`, pass the same variable:

```bash
cd /path/to/private/worktree
CLANGD_INDEX_SERVER_ADDRESS=127.0.0.1:5901 setupworktree -p
```

## VS Code setup

The copied `.vscode/settings.json` should point the clangd extension at the remote-capable clangd:

```json
"clangd.path": "${env:HOME}/opt/llvm-clangd-remote/bin/clangd",
"clangd.arguments": [
  "--compile-commands-dir=${workspaceFolder}/build"
]
```

After mounting a worktree, reload VS Code:

```text
Developer: Reload Window
```

or restart VS Code.

## Validation

Check that the server is running:

```bash
pgrep -af clangd-index-server
```

Open VS Code `Output -> clangd` and confirm:

- clangd is using `~/opt/llvm-clangd-remote/bin/clangd`
- clangd has `linux+grpc` support
- the external index server is `127.0.0.1:5900`, or your chosen custom port
- background indexing is skipped

Then test:

- Go to Definition
- Go to Implementation
- Find References
- Workspace symbol search

The opened files should be inside the current worktree. If navigation opens files in `/home/peach/repos/OrcaSlicer` while you are in a different worktree, the worktree `.clangd` has the wrong `MountPoint`; rerun `mountclangd` from the worktree root.

## Regenerating the index

Regenerate the index when:

- switching the main indexed checkout to a meaningfully different branch
- adding, deleting, or renaming many source/header files
- changing important headers used across the project
- changing CMake options that affect `build/compile_commands.json`
- rebuilding the clangd remote toolchain

Run:

```bash
genindex
```

or:

```bash
genindex -p
```

Then restart the matching `clangdserver`.

## Useful environment variables

Limit indexer CPU usage:

```bash
JOBS=8 genindex
```

Use a different server port:

```bash
CLANGD_INDEX_SERVER_ADDRESS=127.0.0.1:5901 clangdserver
```

Use a custom clang resource directory:

```bash
CLANG_RESOURCE_DIR=/path/to/lib/clang/23 genindex
```

Use a different build directory:

```bash
BUILD_DIR=/path/to/build genindex
```

Use a different config from the multi-config compile database:

```bash
CONFIG=Release genindex
```

The default config is `RelWithDebInfo`.

## Troubleshooting

### `float.h` not found

The clang resource directory is missing or mismatched. Check:

```bash
test -f ~/opt/llvm-clangd-remote/lib/clang/23/include/float.h
```

If that file is missing, copy or reinstall the matching LLVM resource directory for the clangd-indexer version.

### Debug-only Eigen or PCH errors during indexing

The generated script filters the multi-config compile database to `RelWithDebInfo` and strips binary PCH flags. This avoids indexing all three Debug/Release/RelWithDebInfo commands for each file.

If it still fails, inspect:

```bash
.clangd-remote-index/clangd-indexer.log
```

### Server cannot bind to the port

Another server is already using the port. Use another address:

```bash
CLANGD_INDEX_SERVER_ADDRESS=127.0.0.1:5901 clangdserver
```

Then remount the worktree using the same address.

### `setupworktree` overwrites `.vscode`

`cpvscode` intentionally removes the current worktree `.vscode` and copies the main checkout's `.vscode`.

If a worktree has custom settings, back them up before running:

```bash
cp -a .vscode ".vscode.backup.$(date +%Y%m%d%H%M%S)"
```

### Worktree still has heavy indexing

Check the worktree `.clangd` includes:

```yaml
Index:
  Background: Skip
```

Reload VS Code after changing `.clangd`.

### Remote index block appears to be ignored

If clangd does not connect to the server after reload, put the external index block in the user config instead of the worktree `.clangd`.

Linux user config path:

```bash
mkdir -p ~/.config/clangd
$EDITOR ~/.config/clangd/config.yaml
```

Example:

```yaml
If:
  PathMatch: /path/to/worktree/.*
Index:
  Background: Skip
  External:
    Server: 127.0.0.1:5900
    MountPoint: /path/to/worktree/
```

## Official references

- clangd remote index guide: <https://clangd.llvm.org/guides/remote-index>
- clangd remote index design notes: <https://clangd.llvm.org/design/remote-index>
- clangd configuration reference: <https://clangd.llvm.org/config>
- LLVM getting started/build documentation: <https://llvm.org/docs/GettingStarted.html>
