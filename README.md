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

There are two ways to install fuetem. Both install all dependencies automatically.

### Option 1: From AUR (recommended)

```bash
yay -S fuetem-arch
```

This installs fuetem system-wide to `/usr/` and pulls in all dependencies via pacman.

### Option 2: Install from source

```bash
git clone https://github.com/invisi101/fuetem-arch.git
cd fuetem-arch
make install
```

This installs to `~/.local/` by default and runs `pacman -S --needed` for all dependencies. Make sure `~/.local/bin` is in your `PATH`.

To install to a different location:

```bash
make install PREFIX=/usr/local
```

If you already have the dependencies and just want to copy the files:

```bash
make install-files
```

## Uninstall

### AUR package

```bash
yay -R fuetem-arch
```

### Manual install

From anywhere, run:

```bash
fuetem --uninstall
```

This detects where fuetem is installed and removes all its files. You don't need the source repo.

## Usage

```bash
fuetem
```

Select a module from the numbered menu. Some modules require `sudo` and will prompt as needed.

## Module Details

### 1. System Monitor

A real-time TUI dashboard that opens in a new terminal window. It displays CPU load and usage with colour-coded bars, per-sensor temperature readings from `lm_sensors` (green/yellow/red based on thresholds), RAM and swap usage, disk space for `/` and `/home`, the top processes sorted by CPU consumption, battery level with charge rate, health percentage and time remaining, and GPU stats for Nvidia cards via `nvidia-smi`. Press any key to refresh or `q` to quit.

This gives you a single-screen overview of your system's state without needing to remember separate commands for each metric. Useful for diagnosing performance issues, monitoring thermals under load, or keeping an eye on battery drain.

### 2. System Health

Runs a comprehensive health audit of the system:

- **Pacman database integrity** — runs `pacman -Dk` to detect missing files and broken dependencies. A corrupted package database can prevent future installs and updates, so catching this early avoids a broken system.
- **NVMe SMART health** — queries each NVMe drive for its SMART health status and wear level percentage via `smartctl`. SSDs have a finite write endurance; monitoring wear lets you plan replacements before data loss.
- **Btrfs filesystem health** — if root is on btrfs, shows filesystem usage breakdown and scrub status. Btrfs scrubs detect and correct silent data corruption (bitrot), so knowing whether scrubs are running and passing is critical for data integrity.
- **Network interfaces** — lists active network interfaces and their state using `ip link`.
- **Systemd timers** — shows all scheduled timers (the systemd equivalent of cron jobs), so you can verify backups, updates, and other automated tasks are actually scheduled.
- **Previous boot errors** — pulls errors from the last boot's journal, filtering out known harmless noise (ACPI firmware bugs, Intel GPU errors, Bluetooth warnings). This helps you spot real problems that might otherwise be buried in log spam.
- **Kernel and initramfs** — shows the running kernel version and lists boot images. After a kernel update, the initramfs must match or the system won't boot properly.
- **Unmerged pacnew/pacsave files** — finds `.pacnew` and `.pacsave` files in `/etc`. These appear when a package update includes a new config file that conflicts with your customised version. Leaving them unmerged can mean you're missing important config changes or running with stale settings.
- **Kernel reboot check** — compares the running kernel against installed module directories. If they don't match, a reboot is needed to run the updated kernel.

### 3. Service Browser

Gives a structured overview of all systemd services:

- Summary counts of running, enabled, and failed services
- Failed services listed first (these need attention)
- All enabled system services and their states
- All running system services
- Enabled user-level services (things like Syncthing, pipewire, etc.)

Systemd manages everything from networking to audio to display managers. A failed service can silently break functionality — for example, a failed `systemd-resolved` means DNS stops working, or a failed `NetworkManager` means no connectivity. This module surfaces those problems immediately rather than waiting until something visibly breaks.

### 4. Cleanup

Reclaims disk space and detects drift across multiple areas:

- **Journal logs** — trims systemd journal to 200MB. Journals can grow to several gigabytes on long-running systems and provide diminishing returns past a reasonable size.
- **Temporary files** — deletes files in `/tmp` and `/var/tmp` older than 7 days. Temporary files accumulate from crashed applications, build processes, and browser downloads.
- **Coredumps** — removes systemd coredumps. These are memory snapshots from crashed processes and can be hundreds of megabytes each.
- **Pacman cache** — runs `paccache -r` to keep only the 3 most recent versions of each package, and `paccache -ruk0` to remove all cached versions of uninstalled packages. The pacman cache grows indefinitely and can consume tens of gigabytes.
- **Yay cache** — if yay is installed, cleans the AUR build cache which accumulates source tarballs and build artifacts.
- **Flatpak runtimes** — removes unused Flatpak runtimes. Flatpak apps share runtimes, but when all apps using a particular runtime are removed, the runtime itself stays behind.
- **Orphaned packages** — uses `pacman -Qtdq` to find packages that were installed as dependencies but are no longer needed. These accumulate over time as you install and remove software. Lists them and asks for confirmation before removal.
- **Broken symlinks** — scans home directory, `/etc`, and `/usr/local/bin` for dangling symlinks, filtering out expected runtime locks. Broken symlinks can cause confusing errors when applications try to follow them.
- **Chezmoi drift** — if chezmoi is installed, checks whether your live dotfiles have diverged from what chezmoi manages. Untracked drift means changes that won't survive a reinstall or a move to a new machine.
- **Failed systemd services** — flags any services that have entered a failed state.

Shows disk usage before and after, with the total space freed.

### 5. Update Check

Queries official repos via `checkupdates` (a safe, lock-free way to check for updates without running pacman) and AUR updates via `yay -Qua` if yay is installed. Shows package names, current versions, and available versions with counts for each.

Keeping packages up to date is the single most important thing you can do for system security. Most exploits target known vulnerabilities that already have patches available. On a rolling release like Arch, falling behind on updates can also make future updates harder due to accumulated breaking changes.

### 6. Downgrade Helper

Parses `/var/log/pacman.log` to show packages upgraded in the last 7 days, then lets you pick one to downgrade from the local pacman cache at `/var/cache/pacman/pkg`.

Package updates occasionally introduce regressions — a new version of a driver might break hardware support, or a library update might break an application. This module lets you quickly roll back to the previous version using `pacman -U` without needing to find and manually download older packages.

### 7. Integrity Check

A multi-layered security audit:

- **AIDE file integrity** — runs the AIDE intrusion detection system to compare current filesystem state against a known-good baseline. Detects unauthorized modifications to system binaries, libraries, and config files. This is how you catch rootkits and backdoors that modify existing executables.
- **Pacman file integrity** — runs `pacman -Qkk` to verify that files installed by packages still match their original checksums, filtering out expected changes in `/etc`, `/var`, and `/tmp`. Catches corruption and tampering at the package level.
- **arch-audit CVE scanning** — queries the Arch security tracker for known vulnerabilities in your installed packages. Shows which packages have unpatched CVEs so you can prioritise updates.
- **Auditd analysis** — if the Linux audit framework is running, analyses the last 24 hours of audit events for sudo usage (privilege escalation tracking), kernel module loads (could indicate rootkit installation), executables run from `/tmp` (a common attacker technique), and failed login attempts (brute force detection).
- **SUID/SGID scan** — finds files with the setuid or setgid bit set outside standard system directories. SUID binaries run with elevated privileges and are a common target for privilege escalation exploits. Unexpected SUID files outside `/usr/bin`, `/usr/lib`, and `/usr/sbin` are a red flag.
- **World-writable files** — finds files writable by any user outside temporary directories. World-writable files in system paths can be overwritten by any local user, which is a privilege escalation vector.
- **Boot partition permissions** — checks that `/boot` and its contents (including the bootloader random seed) aren't world-readable, accounting for FAT32 partitions where Unix permissions don't apply.

Produces a scored summary (green/yellow/orange/red) based on the number and severity of findings.

### 8. Network Port Scan

- **Local listening ports** — runs `ss -tulpn` to show every TCP and UDP port your system is listening on, with the process name and PID for each. Any port you don't recognise should be investigated — it could be a legitimate service you forgot about or something that shouldn't be there.
- **Interface and subnet detection** — auto-detects your active network interface and subnet using NetworkManager, systemd-networkd, or dhcpcd (with fallback to the default route).
- **LAN scan** — optionally runs an nmap SYN scan of your local network to discover other devices and their open ports. Useful for finding unexpected devices on your network or verifying that your own machines aren't exposing services they shouldn't be.
- **ARP/neighbour table** — shows the current ARP cache, which maps IP addresses to MAC addresses for devices your system has recently communicated with.

### 9. VPN Check

A comprehensive VPN leak audit designed for ProtonVPN with NetShield, but useful for any VPN:

- **Tunnel detection** — checks for active tun/wg/proton interfaces to verify the VPN tunnel is up.
- **Public IP and ASN** — fetches your public IPv4 address and looks up the owning organisation. If you see your ISP instead of your VPN provider, your traffic is leaking.
- **IPv6 exposure** — tests whether IPv6 connectivity exists via DNS, ping, and HTTP. Many VPNs only tunnel IPv4, leaving IPv6 traffic exposed to your ISP.
- **DNS configuration and leak check** — shows your configured nameservers and queries OpenDNS to verify DNS requests are going through the VPN, not leaking to your ISP's resolvers.
- **STUN/WebRTC** — tests whether STUN (used by WebRTC) can reach external servers, which can leak your real IP even through a VPN.
- **Ad/tracker scorecard** — resolves 30+ known ad, tracking, and malware domains and checks whether they're being sinkholed (blocked) by the VPN's DNS filtering. Reports a block rate percentage and flags any domains that are getting through.

Produces timestamped text and CSV reports in the logs directory.

### 10. Secret Scan

Scans every git repository under your home directory for accidentally committed secrets (API keys, passwords, tokens, private keys) using two tools:

- **TruffleHog** — scans git history for high-entropy strings and known secret patterns, distinguishing between verified (confirmed live) and unverified secrets.
- **Gitleaks** — pattern-based scanning using a comprehensive ruleset for common secret formats.

Produces a summary table showing findings per repository and a detailed log file. Accidentally committed secrets are one of the most common causes of security breaches — a single AWS key in a public repo can result in thousands of dollars of charges within hours.

### 11. Verify File Checksum

SHA-256 file verification with convenience features:

- Prompts for a file path (with tab completion)
- Auto-detects SHA-256 hashes in the Wayland clipboard (via `wl-paste`) so you can copy a checksum from a download page and just confirm
- Falls back to manual entry if no clipboard hash is found
- Computes and compares the checksums, reporting match or mismatch

Verifying checksums after downloading ISOs, firmware, or security-critical software confirms the file hasn't been corrupted in transit or tampered with.

## Dependencies

All dependencies are installed automatically by both install methods.

| Package | Used by |
|---------|---------|
| `bash` | Shell |
| `pacman-contrib` | `paccache`, `checkupdates` |
| `bind` | DNS queries, VPN leak check |
| `iproute2` | Network info, port scanning |
| `coreutils` | Core utilities |
| `systemd` | Service browser, journal, timers |
| `smartmontools` | NVMe SMART health checks |
| `nmap` | LAN network scanning |
| `lm_sensors` | Temperature monitoring |
| `arch-audit` | CVE vulnerability scanning |
| `gitleaks` | Git secret scanning |

The following are not in the official repos or have heavier footprints. They are not installed automatically — install them manually if you need the features they enable.

| Package | Used by |
|---------|---------|
| `trufflehog` | Git secret scanning |
| `aide` | File integrity monitoring |
| `audit` | Security event analysis |
| `btrfs-progs` | Btrfs filesystem checks |
| `wl-clipboard` | Clipboard checksum detection |
| `yay` | AUR updates and cache cleanup |
| `chezmoi` | Dotfile drift detection |
| `flatpak` | Flatpak cleanup |

## Logs

Logs are stored in `${XDG_DATA_HOME:-~/.local/share}/fuetem/logs/`.

## License

[GPL-3.0](LICENSE)
