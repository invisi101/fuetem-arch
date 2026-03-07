# fuetem-arch Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Package the arch-maint scripts into a distributable Arch Linux package with both AUR (PKGBUILD) and manual (Makefile) install methods.

**Architecture:** A `bin/fuetem` launcher sources `lib/lib.sh` (shared config: XDG paths, terminal detection) then sources and runs `lib/main.sh` (menu + all functions). Sub-scripts (`vpncheck.sh`, `scan-secrets.sh`, `integrity_check.sh`, `sysmonitor.sh`) are invoked from main.sh via `$FUETEM_LIB_DIR`. The original scripts in `~/dev/scripts/arch-maint/` are untouched — all work happens in `~/dev/fuetem-arch/`.

**Tech Stack:** Bash, GNU Make, makepkg (PKGBUILD)

**Source originals:** `~/dev/scripts/arch-maint/` (read-only reference, do not modify)

---

### Task 1: Initialize git repo and directory structure

**Files:**
- Create: `~/dev/fuetem-arch/bin/` (directory)
- Create: `~/dev/fuetem-arch/lib/` (directory)
- Create: `~/dev/fuetem-arch/assets/` (directory)

**Step 1: Create directory structure**

```bash
cd ~/dev/fuetem-arch
mkdir -p bin lib assets docs/plans
```

**Step 2: Initialize git repo**

```bash
cd ~/dev/fuetem-arch
git init
```

**Step 3: Copy the logo asset**

```bash
cp ~/dev/scripts/arch-maint/arch.png ~/dev/fuetem-arch/assets/
```

**Step 4: Create .gitignore**

Create `~/dev/fuetem-arch/.gitignore`:
```
*.pkg.tar.zst
pkg/
src/
```

**Step 5: Commit**

```bash
cd ~/dev/fuetem-arch
git add .gitignore assets/ docs/
git commit -m "init: project structure, logo asset, and design docs"
```

---

### Task 2: Create lib.sh (shared library)

This is the new shared library that replaces all hardcoded paths and provides terminal detection.

**Files:**
- Create: `lib/lib.sh`

**Step 1: Write lib.sh**

Create `~/dev/fuetem-arch/lib/lib.sh`:
```bash
#!/usr/bin/env bash
# lib.sh — shared configuration and helpers for fuetem

# FUETEM_LIB_DIR is set by the launcher; fallback for direct sourcing
FUETEM_LIB_DIR="${FUETEM_LIB_DIR:-$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)}"

# XDG-compliant data and log directories
FUETEM_DATA_DIR="${XDG_DATA_HOME:-$HOME/.local/share}/fuetem"
FUETEM_LOG_DIR="$FUETEM_DATA_DIR/logs"
mkdir -p "$FUETEM_LOG_DIR"

# open_terminal — launch a command in a new terminal window
# Usage: open_terminal "Title" "class" command [args...]
open_terminal() {
	local title="$1" wm_class="$2"
	shift 2

	# 1. $TERMINAL env var (set by many WMs/DEs)
	if [[ -n "${TERMINAL:-}" ]]; then
		case "$TERMINAL" in
			kitty)       "$TERMINAL" --title "$title" --class "$wm_class" "$@" & ;;
			alacritty)   "$TERMINAL" --title "$title" --class "$wm_class" -e "$@" & ;;
			foot)        "$TERMINAL" --title "$title" --app-id "$wm_class" "$@" & ;;
			wezterm)     "$TERMINAL" start --class "$wm_class" -- "$@" & ;;
			ghostty)     "$TERMINAL" --title="$title" --class="$wm_class" -e "$@" & ;;
			*)           "$TERMINAL" -e "$@" & ;;
		esac
		disown
		return 0
	fi

	# 2. Probe common terminals
	local term
	for term in kitty alacritty foot wezterm ghostty xterm; do
		if command -v "$term" >/dev/null 2>&1; then
			TERMINAL="$term"
			open_terminal "$title" "$wm_class" "$@"
			return $?
		fi
	done

	# 3. Fall back to running in current terminal
	echo "No graphical terminal found — running in current terminal."
	"$@"
}
```

**Step 2: Verify syntax**

```bash
bash -n ~/dev/fuetem-arch/lib/lib.sh
```
Expected: no output (clean parse)

**Step 3: Commit**

```bash
cd ~/dev/fuetem-arch
git add lib/lib.sh
git commit -m "feat: add lib.sh shared library with XDG paths and terminal detection"
```

---

### Task 3: Create main.sh (menu + all inline functions)

Copy `arch.sh` and adapt it: replace `SCRIPT_DIR` with `$FUETEM_LIB_DIR`, replace `$HOME/dev/logs` with `$FUETEM_LOG_DIR`, replace hardcoded `kitty` with `open_terminal()`.

**Files:**
- Create: `lib/main.sh`

**Step 1: Copy arch.sh as base**

```bash
cp ~/dev/scripts/arch-maint/arch.sh ~/dev/fuetem-arch/lib/main.sh
```

**Step 2: Apply modifications to main.sh**

These are the exact changes needed (all other code stays identical):

a) **Remove the shebang and set -euo pipefail** from the top (lines 1-2). The launcher handles these. Replace with a guard comment:
```bash
# main.sh — sourced by the fuetem launcher, do not run directly
```

b) **vpn_check() function** (~line 405-422): Replace the function body:
```bash
vpn_check() {
	echo "🔒 Running VPN check..."
	echo

	local LOG_FILE="$FUETEM_LOG_DIR/vpncheck-$(date +%F).txt"

	if [[ -f "$FUETEM_LIB_DIR/vpncheck.sh" ]]; then
		bash "$FUETEM_LIB_DIR/vpncheck.sh" 2>&1 | tee "$LOG_FILE" || true
		echo
		echo "Log saved to: $LOG_FILE"
	else
		echo "❌ vpncheck.sh missing in $FUETEM_LIB_DIR"
	fi
}
```

c) **secret_scan() function** (~line 427-438): Replace the function body:
```bash
secret_scan() {
	echo "🔍 Running git secret scan..."
	echo

	if [[ -f "$FUETEM_LIB_DIR/scan-secrets.sh" ]]; then
		bash "$FUETEM_LIB_DIR/scan-secrets.sh" || true
	else
		echo "❌ scan-secrets.sh missing in $FUETEM_LIB_DIR"
	fi
}
```

d) **system_monitor() function** (~line 573-579): Replace with:
```bash
system_monitor() {
	echo "📊 Launching System Monitor..."
	open_terminal "System Monitor" "sysmonitor" bash "$FUETEM_LIB_DIR/sysmonitor.sh"
}
```

e) **integrity_check() function** (~line 584-595): Replace with:
```bash
integrity_check() {
	echo "🔒 Running Integrity & Security Check..."
	echo

	if [[ -f "$FUETEM_LIB_DIR/integrity_check.sh" ]]; then
		bash "$FUETEM_LIB_DIR/integrity_check.sh" || true
	else
		echo "❌ integrity_check.sh missing in $FUETEM_LIB_DIR"
	fi
}
```

**Step 3: Verify syntax**

```bash
bash -n ~/dev/fuetem-arch/lib/main.sh
```
Expected: no output (clean parse)

**Step 4: Commit**

```bash
cd ~/dev/fuetem-arch
git add lib/main.sh
git commit -m "feat: add main.sh menu adapted from arch.sh with lib.sh integration"
```

---

### Task 4: Adapt sub-scripts (vpncheck, scan-secrets, integrity_check, sysmonitor)

Copy each sub-script and fix hardcoded paths.

**Files:**
- Create: `lib/vpncheck.sh`
- Create: `lib/scan-secrets.sh`
- Create: `lib/integrity_check.sh`
- Create: `lib/sysmonitor.sh`

**Step 1: Copy vpncheck.sh and fix log dir**

```bash
cp ~/dev/scripts/arch-maint/vpncheck.sh ~/dev/fuetem-arch/lib/vpncheck.sh
```

Change in `lib/vpncheck.sh` line 27:
```bash
# Old:
OUTDIR="$HOME/dev/logs"
# New:
OUTDIR="${FUETEM_LOG_DIR:-${XDG_DATA_HOME:-$HOME/.local/share}/fuetem/logs}"
```

**Step 2: Copy scan-secrets.sh and fix hardcoded paths**

```bash
cp ~/dev/scripts/arch-maint/scan-secrets.sh ~/dev/fuetem-arch/lib/scan-secrets.sh
```

Changes in `lib/scan-secrets.sh`:

Line 7-8 — replace hardcoded tool paths with command -v lookups:
```bash
# Old:
TRUFFLEHOG="$HOME/.local/bin/trufflehog"
GITLEAKS="/usr/bin/gitleaks"
# New:
TRUFFLEHOG="$(command -v trufflehog 2>/dev/null || true)"
GITLEAKS="$(command -v gitleaks 2>/dev/null || true)"

if [[ -z "$TRUFFLEHOG" ]]; then
    echo "❌ trufflehog not found in PATH. Install it first."
    exit 1
fi
if [[ -z "$GITLEAKS" ]]; then
    echo "❌ gitleaks not found in PATH. Install it first."
    exit 1
fi
```

Line 9 — replace hardcoded log dir:
```bash
# Old:
LOG_DIR="$HOME/dev/logs"
# New:
LOG_DIR="${FUETEM_LOG_DIR:-${XDG_DATA_HOME:-$HOME/.local/share}/fuetem/logs}"
```

**Step 3: Copy integrity_check.sh (no changes needed)**

```bash
cp ~/dev/scripts/arch-maint/integrity_check.sh ~/dev/fuetem-arch/lib/integrity_check.sh
```

No hardcoded paths to fix — it uses `/tmp/aide_last_check.log` which is fine.

**Step 4: Copy sysmonitor.sh (no changes needed)**

```bash
cp ~/dev/scripts/arch-maint/sysmonitor.sh ~/dev/fuetem-arch/lib/sysmonitor.sh
```

No hardcoded paths — it reads from /proc and /sys.

**Step 5: Verify syntax of all scripts**

```bash
for f in ~/dev/fuetem-arch/lib/*.sh; do bash -n "$f" && echo "OK: $f"; done
```
Expected: OK for all 5 files

**Step 6: Commit**

```bash
cd ~/dev/fuetem-arch
git add lib/vpncheck.sh lib/scan-secrets.sh lib/integrity_check.sh lib/sysmonitor.sh
git commit -m "feat: add sub-scripts with hardcoded paths replaced"
```

---

### Task 5: Create the launcher (bin/fuetem)

**Files:**
- Create: `bin/fuetem`

**Step 1: Write the launcher**

Create `~/dev/fuetem-arch/bin/fuetem`:
```bash
#!/usr/bin/env bash
set -euo pipefail

# Resolve FUETEM_LIB_DIR: installed location or development layout
if [[ -f "/usr/lib/fuetem/lib.sh" ]]; then
	FUETEM_LIB_DIR="/usr/lib/fuetem"
elif [[ -f "${HOME}/.local/lib/fuetem/lib.sh" ]]; then
	FUETEM_LIB_DIR="${HOME}/.local/lib/fuetem"
else
	# Development: running from repo checkout
	FUETEM_LIB_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")/../lib" && pwd)"
fi
export FUETEM_LIB_DIR

source "$FUETEM_LIB_DIR/lib.sh"
source "$FUETEM_LIB_DIR/main.sh"
```

**Step 2: Make executable**

```bash
chmod +x ~/dev/fuetem-arch/bin/fuetem
```

**Step 3: Verify syntax**

```bash
bash -n ~/dev/fuetem-arch/bin/fuetem
```
Expected: no output (clean parse)

**Step 4: Commit**

```bash
cd ~/dev/fuetem-arch
git add bin/fuetem
git commit -m "feat: add fuetem launcher with install-path detection"
```

---

### Task 6: Create Makefile

**Files:**
- Create: `Makefile`

**Step 1: Write the Makefile**

Create `~/dev/fuetem-arch/Makefile`:
```makefile
PREFIX    ?= $(HOME)/.local
BINDIR    = $(PREFIX)/bin
LIBDIR    = $(PREFIX)/lib/fuetem
SHAREDIR  = $(PREFIX)/share/fuetem

.PHONY: install uninstall

install:
	install -d $(BINDIR) $(LIBDIR) $(SHAREDIR)
	install -m 755 bin/fuetem $(BINDIR)/fuetem
	install -m 644 lib/lib.sh $(LIBDIR)/lib.sh
	install -m 644 lib/main.sh $(LIBDIR)/main.sh
	install -m 755 lib/vpncheck.sh $(LIBDIR)/vpncheck.sh
	install -m 755 lib/scan-secrets.sh $(LIBDIR)/scan-secrets.sh
	install -m 755 lib/integrity_check.sh $(LIBDIR)/integrity_check.sh
	install -m 755 lib/sysmonitor.sh $(LIBDIR)/sysmonitor.sh
	install -m 644 assets/arch.png $(SHAREDIR)/arch.png
	@echo ""
	@echo "Installed to $(PREFIX). Make sure $(BINDIR) is in your PATH."

uninstall:
	rm -f $(BINDIR)/fuetem
	rm -rf $(LIBDIR)
	rm -rf $(SHAREDIR)
	@echo "Uninstalled fuetem from $(PREFIX)."
```

**Step 2: Verify Makefile parses**

```bash
cd ~/dev/fuetem-arch && make -n install
```
Expected: prints the install commands without running them (dry run)

**Step 3: Commit**

```bash
cd ~/dev/fuetem-arch
git add Makefile
git commit -m "feat: add Makefile for user-local install/uninstall"
```

---

### Task 7: Create PKGBUILD

**Files:**
- Create: `PKGBUILD`

**Step 1: Write the PKGBUILD**

Create `~/dev/fuetem-arch/PKGBUILD`:
```bash
# Maintainer: invisi101 <https://github.com/invisi101>
pkgname=fuetem-arch
pkgver=1.0.0
pkgrel=1
pkgdesc="Arch Linux system maintenance console — cleanup, health, security, monitoring"
arch=('any')
url="https://github.com/invisi101/fuetem-arch"
license=('GPL-3.0-only')
depends=('bash' 'pacman-contrib' 'bind' 'iproute2' 'coreutils' 'systemd')
optdepends=(
  'smartmontools: NVMe SMART health checks'
  'nmap: LAN network scanning'
  'lm_sensors: temperature monitoring in system monitor'
  'btrfs-progs: btrfs filesystem checks'
  'wl-clipboard: clipboard-based checksum detection'
  'arch-audit: CVE vulnerability scanning'
  'aide: file integrity monitoring'
  'audit: auditd security event analysis'
  'gitleaks: git secret scanning'
  'trufflehog: git secret scanning'
  'yay: AUR update checking and cache cleanup'
  'chezmoi: dotfile drift detection'
  'flatpak: flatpak cleanup support'
  'kitty: terminal emulator for system monitor window'
  'alacritty: terminal emulator for system monitor window'
  'foot: terminal emulator for system monitor window'
)
source=("${pkgname}-${pkgver}.tar.gz::${url}/archive/v${pkgver}.tar.gz")
sha256sums=('SKIP')

package() {
  cd "${pkgname}-${pkgver}"

  install -Dm755 bin/fuetem "$pkgdir/usr/bin/fuetem"

  install -Dm644 lib/lib.sh "$pkgdir/usr/lib/fuetem/lib.sh"
  install -Dm644 lib/main.sh "$pkgdir/usr/lib/fuetem/main.sh"
  install -Dm755 lib/vpncheck.sh "$pkgdir/usr/lib/fuetem/vpncheck.sh"
  install -Dm755 lib/scan-secrets.sh "$pkgdir/usr/lib/fuetem/scan-secrets.sh"
  install -Dm755 lib/integrity_check.sh "$pkgdir/usr/lib/fuetem/integrity_check.sh"
  install -Dm755 lib/sysmonitor.sh "$pkgdir/usr/lib/fuetem/sysmonitor.sh"

  install -Dm644 assets/arch.png "$pkgdir/usr/share/fuetem/arch.png"
  install -Dm644 LICENSE "$pkgdir/usr/share/licenses/${pkgname}/LICENSE"
}
```

**Step 2: Commit**

```bash
cd ~/dev/fuetem-arch
git add PKGBUILD
git commit -m "feat: add PKGBUILD for AUR distribution"
```

---

### Task 8: Create LICENSE and README.md

**Files:**
- Create: `LICENSE`
- Create: `README.md`

**Step 1: Write GPL-3.0 LICENSE file**

Download or write the GPL-3.0 license text to `~/dev/fuetem-arch/LICENSE`.

**Step 2: Write README.md**

Create `~/dev/fuetem-arch/README.md` covering:
- Project name, one-line description, screenshot/banner (assets/arch.png)
- Feature list (all 11 menu items)
- Install methods: AUR (`yay -S fuetem-arch`) and manual (`make install`)
- Dependencies table (required vs optional)
- Usage: just run `fuetem`
- Uninstall instructions
- License

**Step 3: Commit**

```bash
cd ~/dev/fuetem-arch
git add LICENSE README.md
git commit -m "docs: add GPL-3.0 license and README"
```

---

### Task 9: Shellcheck validation and final smoke test

**Step 1: Run shellcheck on all scripts**

```bash
cd ~/dev/fuetem-arch
shellcheck bin/fuetem lib/*.sh
```
Expected: no errors. Fix any warnings that appear.

**Step 2: Dry-run the launcher from the repo**

```bash
cd ~/dev/fuetem-arch
bash -x bin/fuetem <<< "12"
```
Expected: sources lib.sh and main.sh, prints banner + insult, shows menu, exits on option 12.

**Step 3: Verify Makefile install to a temp prefix**

```bash
cd ~/dev/fuetem-arch
make install PREFIX=/tmp/fuetem-test
ls -la /tmp/fuetem-test/bin/fuetem /tmp/fuetem-test/lib/fuetem/ /tmp/fuetem-test/share/fuetem/
rm -rf /tmp/fuetem-test
```
Expected: all files installed to the right locations.

**Step 4: Fix any issues found, commit**

```bash
cd ~/dev/fuetem-arch
git add -A
git commit -m "fix: address shellcheck and smoke test findings"
```
(Only if there were changes to make.)

---

### Task 10: Push to GitHub

**Step 1: Create remote and push**

```bash
cd ~/dev/fuetem-arch
git remote add origin git@github.com:invisi101/fuetem-arch.git
git branch -M main
git push -u origin main
```

**Step 2: Verify on GitHub**

```bash
gh repo view invisi101/fuetem-arch --web
```
