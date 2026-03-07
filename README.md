# fuetem-arch

**Arch Linux system maintenance console** — a menu-driven TUI that brings together cleanup, health checks, security auditing, VPN leak testing, secret scanning, and real-time system monitoring.

![Arch Banner](assets/arch.png)

## Features

| # | Module | Description |
|---|--------|-------------|
| 1 | **System Monitor** | Real-time TUI dashboard — CPU, temps, RAM, disk, processes, battery, GPU |
| 2 | **System Health** | Pacman DB integrity, NVMe SMART, btrfs, boot errors, kernel reboot check |
| 3 | **Service Browser** | Systemd service overview — running, enabled, failed, user services |
| 4 | **Cleanup** | Journal, tmp, pacman/yay cache, flatpak, orphans, broken symlinks, chezmoi drift |
| 5 | **Update Check** | Official repo + AUR update summary |
| 6 | **Downgrade Helper** | Browse recent upgrades, pick a cached version to roll back |
| 7 | **Integrity Check** | AIDE, pacman file integrity, arch-audit CVEs, auditd analysis, SUID/SGID scan |
| 8 | **Network Port Scan** | Local listening ports + optional nmap LAN sweep |
| 9 | **VPN Check** | ProtonVPN leak audit — IP, DNS, IPv6, STUN, ad/tracker blocking scorecard |
| 10 | **Secret Scan** | TruffleHog + Gitleaks across all local git repos |
| 11 | **Verify File Checksum** | SHA-256 verification with clipboard auto-detect |

## Install

### From AUR

```bash
yay -S fuetem-arch
```

### Manual install

```bash
git clone https://github.com/invisi101/fuetem-arch.git
cd fuetem-arch
make install
```

This installs to `~/.local/` by default. Override with `make install PREFIX=/usr/local`.

### Uninstall

```bash
make uninstall
```

## Dependencies

### Required

| Package | Provides |
|---------|----------|
| `bash` | Shell |
| `pacman-contrib` | `paccache`, `checkupdates` |
| `bind` | `dig` |
| `iproute2` | `ip`, `ss` |
| `coreutils` | Core utilities |
| `systemd` | `systemctl`, `journalctl` |

### Optional

| Package | Used by |
|---------|---------|
| `smartmontools` | NVMe SMART health checks |
| `nmap` | LAN network scanning |
| `lm_sensors` | Temperature monitoring |
| `btrfs-progs` | Btrfs filesystem checks |
| `wl-clipboard` | Clipboard checksum detection |
| `arch-audit` | CVE vulnerability scanning |
| `aide` | File integrity monitoring |
| `audit` | Auditd security analysis |
| `gitleaks` | Git secret scanning |
| `trufflehog` | Git secret scanning |
| `yay` | AUR updates and cache cleanup |
| `chezmoi` | Dotfile drift detection |
| `flatpak` | Flatpak cleanup |
| `kitty` / `alacritty` / `foot` | Terminal for system monitor window |

## Usage

```bash
fuetem
```

Select a module from the menu. That's it.

## Logs

Logs are stored in `${XDG_DATA_HOME:-~/.local/share}/fuetem/logs/`.

## License

[GPL-3.0](LICENSE)
