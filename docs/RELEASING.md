# Releasing fuetem-arch

This document covers everything needed to push a new version of fuetem-arch to GitHub and the AUR.

## Prerequisites

- SSH key for AUR: `~/.ssh/id_ed25519_aur` (has a passphrase)
- SSH config (`~/.ssh/config`) maps `aur.archlinux.org` to that key
- GitHub remote uses HTTPS: `https://github.com/invisi101/fuetem-arch.git`

## Step-by-step release process

### 1. Make your changes

Edit whatever you need in `~/dev/fuetem-arch/`. Test locally with:

```bash
cd ~/dev/fuetem-arch
bash -n bin/fuetem lib/*.sh        # syntax check all scripts
FUETEM_LIB_DIR="$(pwd)/lib" bash bin/fuetem  # run from dev checkout
```

Note: if an older version is installed to `~/.local/` or `/usr/`, the launcher will pick that up instead of your dev copy. Use the `FUETEM_LIB_DIR` override above, or uninstall first.

### 2. Commit and push to GitHub

```bash
cd ~/dev/fuetem-arch
git add <changed files>
git commit -m "description of changes"
git push origin main
```

### 3. Decide the new version number

Look at the current version:

```bash
grep pkgver PKGBUILD
```

Bump it: patch (1.2.0 → 1.2.1) for fixes, minor (1.2.0 → 1.3.0) for new features, major (1.2.0 → 2.0.0) for breaking changes.

### 4. Tag the release on GitHub

```bash
git tag v<NEW_VERSION>
git push origin v<NEW_VERSION>
```

Example:

```bash
git tag v1.3.0
git push origin v1.3.0
```

### 5. Get the SHA256 of the new tarball

GitHub generates a tarball from the tag. You need its hash for the PKGBUILD:

```bash
curl -sL https://github.com/invisi101/fuetem-arch/archive/v<NEW_VERSION>.tar.gz | sha256sum
```

Copy the hash (the first 64 characters).

### 6. Update the PKGBUILD

Edit `PKGBUILD` and change two lines:

```
pkgver=<NEW_VERSION>
sha256sums=('<THE_NEW_HASH>')
```

Leave `pkgrel=1` (only bump pkgrel if you're re-releasing the same version with packaging-only changes).

### 7. Regenerate .SRCINFO

```bash
makepkg --printsrcinfo > .SRCINFO
```

### 8. Commit and push the PKGBUILD update

```bash
git add PKGBUILD .SRCINFO
git commit -m "Release v<NEW_VERSION>: brief description"
git push origin main
```

### 9. Push to the AUR

The AUR is a separate git repo that only contains PKGBUILD and .SRCINFO. You need your SSH key loaded for this step.

Run this in your terminal (not in Claude — the key has a passphrase that needs interactive input):

```bash
eval "$(ssh-agent -s)" && ssh-add ~/.ssh/id_ed25519_aur && rm -rf /tmp/fuetem-aur && git clone ssh://aur@aur.archlinux.org/fuetem-arch.git /tmp/fuetem-aur && cp ~/dev/fuetem-arch/PKGBUILD ~/dev/fuetem-arch/.SRCINFO /tmp/fuetem-aur/ && cd /tmp/fuetem-aur && git add PKGBUILD .SRCINFO && git commit -m "Update to v<NEW_VERSION>: brief description" && git push
```

It will ask for:
1. Your SSH key passphrase (on the `ssh-add` step)
2. Confirmation to overwrite PKGBUILD — type `y`
3. Confirmation to overwrite .SRCINFO — type `y`

### 10. Verify

```bash
# Check AUR has the new version
curl -s "https://aur.archlinux.org/rpc/v5/info?arg[]=fuetem-arch" | python3 -c "import sys,json; print(json.load(sys.stdin)['results'][0]['Version'])"

# Install it (--rebuild forces fresh build if yay has a cached old version)
yay -S fuetem-arch --rebuild
```

## Quick reference (copy-paste version)

Replace `X.Y.Z` with your new version and `DESCRIPTION` with a brief summary:

```bash
# In ~/dev/fuetem-arch after committing your changes:
git push origin main
git tag vX.Y.Z && git push origin vX.Y.Z
curl -sL https://github.com/invisi101/fuetem-arch/archive/vX.Y.Z.tar.gz | sha256sum
# Edit PKGBUILD: pkgver=X.Y.Z and sha256sums=('...')
makepkg --printsrcinfo > .SRCINFO
git add PKGBUILD .SRCINFO && git commit -m "Release vX.Y.Z: DESCRIPTION" && git push origin main
```

Then paste in terminal for the AUR push:

```bash
eval "$(ssh-agent -s)" && ssh-add ~/.ssh/id_ed25519_aur && rm -rf /tmp/fuetem-aur && git clone ssh://aur@aur.archlinux.org/fuetem-arch.git /tmp/fuetem-aur && cp ~/dev/fuetem-arch/PKGBUILD ~/dev/fuetem-arch/.SRCINFO /tmp/fuetem-aur/ && cd /tmp/fuetem-aur && git add PKGBUILD .SRCINFO && git commit -m "Update to vX.Y.Z: DESCRIPTION" && git push
```

## Troubleshooting

### SSH permission denied on AUR push

Your SSH agent doesn't have the key loaded. Run:

```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519_aur
```

### yay still installs the old version after AUR update

Yay caches builds. Force a fresh build:

```bash
yay -S fuetem-arch --rebuild
```

### sha256sum doesn't match after tagging

You probably tagged before pushing the latest commit. The tarball is generated from whatever the tag points to. Delete and retag:

```bash
git tag -d vX.Y.Z
git push origin :refs/tags/vX.Y.Z
git tag vX.Y.Z
git push origin vX.Y.Z
```

Then re-fetch the sha256sum.
