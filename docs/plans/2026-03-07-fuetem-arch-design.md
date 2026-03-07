# fuetem-arch — Package Design

## Overview

Arch Linux system maintenance console. A collection of bash scripts providing
a menu-driven TUI for cleanup, health checks, security auditing, VPN leak
testing, secret scanning, and real-time system monitoring.

## Repository

https://github.com/invisi101/fuetem-arch

## Install Layout

### PKGBUILD (system-wide)

```
/usr/lib/fuetem/
├── lib.sh                 # shared helpers: colors, XDG paths, terminal detection
├── main.sh                # menu + inline functions
├── sysmonitor.sh
├── vpncheck.sh
├── scan-secrets.sh
└── integrity_check.sh

/usr/bin/fuetem             # launcher
/usr/share/fuetem/arch.png  # assets
```

### Makefile (user-local)

```
~/.local/bin/fuetem
~/.local/lib/fuetem/
~/.local/share/fuetem/
```

## Shared Library (lib.sh)

Provides:
- `FUETEM_LIB_DIR` — set at install time, points to lib directory
- `FUETEM_LOG_DIR` — `${XDG_DATA_HOME:-$HOME/.local/share}/fuetem/logs`
- `FUETEM_DATA_DIR` — `${XDG_DATA_HOME:-$HOME/.local/share}/fuetem`
- `open_terminal()` — checks `$TERMINAL`, probes kitty/alacritty/foot/wezterm/xterm, falls back to current terminal

## Changes from Original Scripts

1. New `lib.sh` with shared config and terminal detection
2. Hardcoded `$HOME/dev/logs` replaced with `$FUETEM_LOG_DIR`
3. Hardcoded `$HOME/.local/bin/trufflehog` and `/usr/bin/gitleaks` replaced with `command -v`
4. Hardcoded `kitty` in system_monitor() replaced with `open_terminal()` from lib.sh
5. `SCRIPT_DIR` references replaced with `$FUETEM_LIB_DIR`

All original logic, menu structure, insults, and behavior preserved exactly.

## Dependencies

### Hard (depends)
- pacman-contrib (paccache, checkupdates)
- bind (dig)
- iproute2 (ip, ss)

### Soft (optdepends)
- smartmontools, nmap, lm_sensors, btrfs-progs
- wl-clipboard, arch-audit, aide, audit
- gitleaks, trufflehog, yay, kitty

## License

GPL-3.0-only
