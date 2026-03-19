---
name: rust-cli-scaffold
description: >
  Add GitHub Actions CI + release workflow + curl-pipe installer to any Rust CLI project.
  Generates only the minimal drop-in files: .github/workflows/ci.yml, .github/workflows/release.yml,
  and install.sh. 
  Works for both single-binary repos and multi-crate Cargo workspaces. Use this skill whenever
  the user wants to add CI/CD, a release pipeline, or a curl install script to a Rust project,
  asks for "github actions for rust", "release workflow rust", "cross-compile rust ci",
  "install.sh for rust tool", or wants to wire up a new or existing Rust CLI for distribution.
---

# Rust CLI Scaffold Skill


## What Gets Generated

```
<project>/
├── .github/workflows/
│   ├── ci.yml        ← fmt + clippy + test, 3-OS matrix
│   └── release.yml   ← cross-compile 5 targets on vX.Y.Z tag
└── install.sh        ← production-grade curl-pipe installer
```

Workspace variant: same structure, `release.yml` routes by tag prefix.

---

## Required Inputs

| Input | Example |
|---|---|
| `bin_name` / `crates` | `linehash` or `smart-grep linehash scope why` |
| `github_username` | `quangdang46` |
| `repo_name` | same as project folder |

Multiple crate names → workspace mode. Single name → single binary.

---

## ci.yml

```yaml
name: CI
on:
  push:
    branches: [main, dev]
  pull_request:
    branches: [main]
env:
  CARGO_TERM_COLOR: always
jobs:
  check:
    name: Check (${{ matrix.os }})
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with:
          components: rustfmt, clippy
      - uses: Swatinem/rust-cache@v2
      - run: cargo fmt --all -- --check
      - run: cargo clippy --all-targets --all-features -- -D warnings
      - run: cargo test --all-features   # --workspace for workspaces
```

---

## release.yml

Trigger: `vX.Y.Z` tag (single binary) or `<crate>-vX.Y.Z` / `release-all-vX.Y.Z` (workspace).

Build matrix (5 targets):

| Target | Runner | Method | Suffix |
|---|---|---|---|
| `x86_64-unknown-linux-musl` | ubuntu | `cross` | `linux-x86_64` |
| `aarch64-unknown-linux-musl` | ubuntu | `cross` | `linux-aarch64` |
| `x86_64-apple-darwin` | macos | native | `macos-x86_64` |
| `aarch64-apple-darwin` | macos | native | `macos-aarch64` |
| `x86_64-pc-windows-msvc` | windows | native | `windows-x86_64` |

Jobs:
1. **build** — parallel matrix → `.tar.gz` (Unix) or `.zip` (Windows) per target + `.sha256` sidecar
2. **release** — download all artifacts → attach to GitHub Release via `softprops/action-gh-release@v2`

Key flags: `permissions: contents: write`, `generate_release_notes: true`

---

## install.sh — Production Patterns

Model after `beads_rust` by Dicklesworthstone. Always include these patterns:

### Safety header
```bash
set -euo pipefail
umask 022
```

### Configuration block at top — all tunables in one place
```bash
BINARY_NAME="<bin>"          # or <bin>.exe on Windows
OWNER="<github_user>"
REPO="<repo>"
DEST="${DEST:-$HOME/.local/bin}"
QUIET=0
EASY=0                       # auto-update PATH in rc files
VERIFY=0                     # run self-test after install
FROM_SOURCE=0
UNINSTALL=0
MAX_RETRIES=3
DOWNLOAD_TIMEOUT=120
```

### Argument parsing — support both `--flag value` and `--flag=value`
```bash
while [ $# -gt 0 ]; do
    case "$1" in
        --dest)    DEST="$2";        shift 2;;
        --dest=*)  DEST="${1#*=}";   shift;;
        --version) VERSION="$2";     shift 2;;
        --system)  DEST="/usr/local/bin"; shift;;
        --easy-mode) EASY=1;         shift;;
        --verify)    VERIFY=1;       shift;;
        --from-source) FROM_SOURCE=1; shift;;
        --quiet|-q)  QUIET=1;        shift;;
        --uninstall) UNINSTALL=1;    shift;;
        -h|--help)   usage;;
        *) shift;;
    esac
done
```

### Platform detection — map to asset suffix
```bash
detect_platform() {
    local os arch
    case "$(uname -s)" in
        Linux*)  os="linux" ;;
        Darwin*) os="darwin" ;;
        MINGW*|MSYS*|CYGWIN*) os="windows" ;;
        *) die "Unsupported OS: $(uname -s)" ;;
    esac
    case "$(uname -m)" in
        x86_64|amd64)   arch="x86_64" ;;
        aarch64|arm64)  arch="aarch64" ;;
        *) die "Unsupported arch: $(uname -m)" ;;
    esac
    echo "${os}_${arch}"
}
```

Then map `linux_x86_64` → `<bin>-linux-x86_64.tar.gz` etc.

### Version resolution with fallback chain
```bash
resolve_version() {
    [ -n "$VERSION" ] && return 0
    # 1st try: GitHub API
    VERSION=$(curl -fsSL \
        --connect-timeout 10 --max-time 30 \
        -H "Accept: application/vnd.github.v3+json" \
        "https://api.github.com/repos/${OWNER}/${REPO}/releases/latest" \
        2>/dev/null | grep '"tag_name":' | sed -E 's/.*"([^"]+)".*/\1/')
    # 2nd fallback: redirect trick
    if [ -z "$VERSION" ]; then
        VERSION=$(curl -fsSL -o /dev/null -w '%{url_effective}' \
            "https://github.com/${OWNER}/${REPO}/releases/latest" \
            2>/dev/null | sed -E 's|.*/tag/||')
    fi
    [[ "$VERSION" =~ ^v[0-9] ]] || die "Could not resolve version"
}
```

### Download with retry + resume + proxy support
```bash
download_file() {
    local url="$1" dest="$2"
    local partial="${dest}.part"
    local attempt=0
    while [ $attempt -lt $MAX_RETRIES ]; do
        attempt=$((attempt + 1))
        curl -fL \
            --connect-timeout 30 \
            --max-time "$DOWNLOAD_TIMEOUT" \
            --retry 2 \
            $( [ -s "$partial" ] && echo "--continue-at -" ) \
            $( [ "$QUIET" -eq 0 ] && [ -t 2 ] && echo "--progress-bar" || echo "-sS" ) \
            -o "$partial" "$url" && mv -f "$partial" "$dest" && return 0
        [ $attempt -lt $MAX_RETRIES ] && { log_warn "Retrying in 3s..."; sleep 3; }
    done
    return 1
}
```

### Checksum verification
```bash
# Download sidecar checksum: <archive>.sha256
if download_file "${url}.sha256" "$TMP/checksum.sha256" 2>/dev/null; then
    expected=$(awk '{print $1}' "$TMP/checksum.sha256")
    actual=$(sha256sum "$TMP/$archive" 2>/dev/null || shasum -a 256 "$TMP/$archive" | awk '{print $1}')
    [ "$expected" = "$actual" ] || die "Checksum mismatch"
fi
```

### Atomic binary installation
```bash
install_binary_atomic() {
    local src="$1" dest="$2"
    local tmp="${dest}.tmp.$$"
    install -m 0755 "$src" "$tmp"
    mv -f "$tmp" "$dest" || { rm -f "$tmp"; die "Failed to install binary"; }
}
```

### Locking — prevents concurrent installs
```bash
LOCK_DIR="/tmp/${BINARY_NAME}-install.lock.d"
acquire_lock() {
    if mkdir "$LOCK_DIR" 2>/dev/null; then
        echo $$ > "$LOCK_DIR/pid"
        return 0
    fi
    die "Another install is running. If stuck: rm -rf $LOCK_DIR"
}
cleanup() { rm -rf "$TMP" "$LOCK_DIR"; }
trap cleanup EXIT
```

### Uninstall flag
```bash
do_uninstall() {
    rm -f "$DEST/$BINARY_NAME"
    # Remove PATH lines added by easy-mode from ~/.bashrc, ~/.zshrc
    for rc in "$HOME/.bashrc" "$HOME/.zshrc"; do
        [ -f "$rc" ] && sed -i "/${BINARY_NAME} installer/d" "$rc" 2>/dev/null || true
    done
    log_success "Uninstalled"
    exit 0
}
[ "$UNINSTALL" -eq 1 ] && do_uninstall
```

### Easy-mode PATH update
```bash
maybe_add_path() {
    case ":$PATH:" in *":$DEST:"*) return 0;; esac
    if [ "$EASY" -eq 1 ]; then
        for rc in "$HOME/.zshrc" "$HOME/.bashrc"; do
            [ -f "$rc" ] && [ -w "$rc" ] || continue
            grep -qF "$DEST" "$rc" && continue
            printf '\nexport PATH="%s:$PATH"  # %s installer\n' "$DEST" "$BINARY_NAME" >> "$rc"
        done
        log_warn "PATH updated — restart shell or: export PATH=\"$DEST:\$PATH\""
    else
        log_warn "Add to PATH: export PATH=\"$DEST:\$PATH\""
    fi
}
```

### From-source fallback
When binary download fails, fall back to:
```bash
build_from_source() {
    command -v cargo >/dev/null || die "Rust/cargo not found. Install: https://rustup.rs"
    git clone --depth 1 "https://github.com/${OWNER}/${REPO}.git" "$TMP/src"
    (cd "$TMP/src" && CARGO_TARGET_DIR="$TMP/target" cargo build --release)
    install_binary_atomic "$TMP/target/release/$BINARY_NAME" "$DEST/$BINARY_NAME"
}
```

### Summary on success
```bash
print_summary() {
    echo ""
    echo "✓ ${BINARY_NAME} installed → $DEST/$BINARY_NAME"
    echo "  Version: $("$DEST/$BINARY_NAME" --version 2>/dev/null || echo 'unknown')"
    echo ""
    echo "  Quick start:"
    echo "    $BINARY_NAME --help"
}
```

### curl | bash safety wrapper
Always end main() call with the buffering wrapper — protects against truncated downloads:
```bash
if [[ "${BASH_SOURCE[0]:-}" == "${0:-}" ]] || [[ -z "${BASH_SOURCE[0]:-}" ]]; then
    { main "$@"; }
fi
```

---

## install.sh Full Skeleton

```bash
#!/usr/bin/env bash
set -euo pipefail
umask 022

# === Config ===
BINARY_NAME="<BIN>"
OWNER="<USER>"
REPO="<REPO>"
DEST="${DEST:-$HOME/.local/bin}"
VERSION="${VERSION:-}"
QUIET=0; EASY=0; VERIFY=0; FROM_SOURCE=0; UNINSTALL=0
MAX_RETRIES=3; DOWNLOAD_TIMEOUT=120
LOCK_DIR="/tmp/${BINARY_NAME}-install.lock.d"
TMP=""

# === Logging ===
log_info()    { [ "$QUIET" -eq 1 ] && return; echo "[${BINARY_NAME}] $*" >&2; }
log_warn()    { echo "[${BINARY_NAME}] WARN: $*" >&2; }
log_success() { [ "$QUIET" -eq 1 ] && return; echo "✓ $*" >&2; }
die()         { echo "ERROR: $*" >&2; exit 1; }

# === Cleanup & lock ===
cleanup() { rm -rf "$TMP" "$LOCK_DIR" 2>/dev/null || true; }
trap cleanup EXIT
acquire_lock() {
    mkdir "$LOCK_DIR" 2>/dev/null || die "Another install running. rm -rf $LOCK_DIR"
    echo $$ > "$LOCK_DIR/pid"
}

# === Args ===
while [ $# -gt 0 ]; do
    case "$1" in
        --dest)       DEST="$2";   shift 2;;
        --dest=*)     DEST="${1#*=}"; shift;;
        --version)    VERSION="$2"; shift 2;;
        --version=*)  VERSION="${1#*=}"; shift;;
        --system)     DEST="/usr/local/bin"; shift;;
        --easy-mode)  EASY=1;      shift;;
        --verify)     VERIFY=1;    shift;;
        --from-source) FROM_SOURCE=1; shift;;
        --quiet|-q)   QUIET=1;     shift;;
        --uninstall)  UNINSTALL=1; shift;;
        *) shift;;
    esac
done

# === Uninstall ===
if [ "$UNINSTALL" -eq 1 ]; then
    rm -f "$DEST/$BINARY_NAME"
    for rc in "$HOME/.bashrc" "$HOME/.zshrc"; do
        [ -f "$rc" ] && sed -i "/${BINARY_NAME} installer/d" "$rc" 2>/dev/null || true
    done
    echo "✓ ${BINARY_NAME} uninstalled"; exit 0
fi

# === Platform ===
detect_platform() {
    local os arch
    case "$(uname -s)" in
        Linux*)  os="linux";;   Darwin*) os="darwin";;
        MINGW*|MSYS*|CYGWIN*) os="windows";;
        *) die "Unsupported OS";;
    esac
    case "$(uname -m)" in
        x86_64|amd64)  arch="x86_64";;
        aarch64|arm64) arch="aarch64";;
        *) die "Unsupported arch";;
    esac
    echo "${os}_${arch}"
}

# === Version ===
resolve_version() {
    [ -n "$VERSION" ] && return 0
    VERSION=$(curl -fsSL --connect-timeout 10 --max-time 30 \
        "https://api.github.com/repos/${OWNER}/${REPO}/releases/latest" 2>/dev/null \
        | grep '"tag_name":' | sed -E 's/.*"([^"]+)".*/\1/') || true
    if ! [[ "$VERSION" =~ ^v[0-9] ]]; then
        VERSION=$(curl -fsSL -o /dev/null -w '%{url_effective}' \
            "https://github.com/${OWNER}/${REPO}/releases/latest" 2>/dev/null \
            | sed -E 's|.*/tag/||') || true
    fi
    [[ "$VERSION" =~ ^v[0-9] ]] || die "Could not resolve version"
    log_info "Latest: $VERSION"
}

# === Download ===
download_file() {
    local url="$1" dest="$2" partial="${2}.part" attempt=0
    while [ $attempt -lt $MAX_RETRIES ]; do
        attempt=$((attempt + 1))
        curl -fL --connect-timeout 30 --max-time "$DOWNLOAD_TIMEOUT" \
             -sS --retry 2 \
             $( [ -s "$partial" ] && echo "--continue-at -") \
             -o "$partial" "$url" \
          && mv -f "$partial" "$dest" && return 0
        [ $attempt -lt $MAX_RETRIES ] && { log_warn "Retry $attempt..."; sleep 3; }
    done
    return 1
}

# === Atomic install ===
install_binary_atomic() {
    local tmp="${2}.tmp.$$"
    install -m 0755 "$1" "$tmp" && mv -f "$tmp" "$2" || { rm -f "$tmp"; die "Install failed"; }
}

# === PATH ===
maybe_add_path() {
    case ":$PATH:" in *":$DEST:"*) return 0;; esac
    if [ "$EASY" -eq 1 ]; then
        for rc in "$HOME/.zshrc" "$HOME/.bashrc"; do
            [ -f "$rc" ] && [ -w "$rc" ] || continue
            grep -qF "$DEST" "$rc" && continue
            printf '\nexport PATH="%s:$PATH"  # %s installer\n' "$DEST" "$BINARY_NAME" >> "$rc"
        done
    fi
    log_warn "Restart shell or: export PATH=\"$DEST:\$PATH\""
}

# === Source build ===
build_from_source() {
    command -v cargo >/dev/null || die "cargo not found — install Rust: https://rustup.rs"
    git clone --depth 1 "https://github.com/${OWNER}/${REPO}.git" "$TMP/src"
    (cd "$TMP/src" && CARGO_TARGET_DIR="$TMP/target" cargo build --release)
    install_binary_atomic "$TMP/target/release/$BINARY_NAME" "$DEST/$BINARY_NAME"
}

# === Main ===
main() {
    acquire_lock
    TMP=$(mktemp -d)
    mkdir -p "$DEST"

    local platform; platform=$(detect_platform)
    log_info "Platform: $platform | Dest: $DEST"

    if [ "$FROM_SOURCE" -eq 0 ]; then
        resolve_version
        local ext="tar.gz"; [[ "$platform" == windows* ]] && ext="zip"
        local archive="${BINARY_NAME}-${VERSION}-${platform}.${ext}"
        local url="https://github.com/${OWNER}/${REPO}/releases/download/${VERSION}/${archive}"

        if download_file "$url" "$TMP/$archive"; then
            # Verify checksum if sidecar exists
            if download_file "${url}.sha256" "$TMP/checksum.sha256" 2>/dev/null; then
                local expected actual
                expected=$(awk '{print $1}' "$TMP/checksum.sha256")
                actual=$(sha256sum "$TMP/$archive" 2>/dev/null | awk '{print $1}' \
                      || shasum -a 256 "$TMP/$archive" | awk '{print $1}')
                [ "$expected" = "$actual" ] || die "Checksum mismatch"
                log_info "Checksum verified"
            fi
            # Extract
            case "$archive" in
                *.tar.gz) tar -xzf "$TMP/$archive" -C "$TMP";;
                *.zip)    unzip -q "$TMP/$archive" -d "$TMP";;
            esac
            local bin; bin=$(find "$TMP" -name "$BINARY_NAME" -type f -perm -111 \
                          2>/dev/null | head -1)
            [ -n "$bin" ] || die "Binary not found after extract"
            install_binary_atomic "$bin" "$DEST/$BINARY_NAME"
        else
            log_warn "Binary download failed — building from source..."
            build_from_source
        fi
    else
        build_from_source
    fi

    maybe_add_path

    [ "$VERIFY" -eq 1 ] && "$DEST/$BINARY_NAME" --version

    echo ""
    echo "✓ $BINARY_NAME installed → $DEST/$BINARY_NAME"
    echo "  $("$DEST/$BINARY_NAME" --version 2>/dev/null || true)"
    echo ""
    echo "  Usage: $BINARY_NAME --help"
}

# curl|bash safety: buffer entire script before executing
if [[ "${BASH_SOURCE[0]:-}" == "${0:-}" ]] || [[ -z "${BASH_SOURCE[0]:-}" ]]; then
    { main "$@"; }
fi
```

---

## Output 
Write only these files:

- `.github/workflows/ci.yml`
- `.github/workflows/release.yml`
- `install.sh`

Then update `README.md` to add or refresh the install command with this exact pattern:

```bash
curl -fsSL "https://raw.githubusercontent.com/<username>/<repo>/main/install.sh?$(date +%s)" | bash
```

End with:
```bash
git tag v0.1.0 && git push origin main --tags
```