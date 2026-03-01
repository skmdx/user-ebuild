# user-ebuild

A lightweight wrapper script to run Gentoo's `ebuild` command as a non-root user using **Bubblewrap (`bwrap`)**.

## Overview

`user-ebuild` creates an isolated sandbox environment using Linux namespaces. It allows you to perform ebuild phases like `unpack`, `compile`, and `package` (creating binary packages) without requiring `sudo` or root privileges on your host system.

## Features

- **Safe Isolation**: The host system's root filesystem is mounted as read-only.
- **Fake Root**: Maps your current user to `uid 0` (root) inside the sandbox to satisfy Portage's permission requirements.
- **Writable Overlays**: Redirects Portage's writable directories (`PORTAGE_TMPDIR`, `DISTDIR`, `PKGDIR`) to a user-owned directory in `/tmp`.
- **User Identity Spoofing**: Automatically generates fake `/etc/passwd` and `/etc/group` files inside the sandbox to prevent `chown` and permission errors.
- **Manifest Support**: Automatically bind-mounts local ebuild directories as writable when running the `manifest` command, provided you have write access to them.

## Prerequisites

- **Bubblewrap**: `sys-apps/bubblewrap` must be installed.
- **Portage**: `sys-apps/portage` must be installed (for the `ebuild` command and configuration files).

## Installation

1. Copy the `user-ebuild` script to your local path (e.g., `~/bin/` or the current directory).
2. Make it executable:
   ```bash
   chmod +x user-ebuild
   ```

## Usage

Run it exactly like the standard `ebuild` command:

```bash
# Unpack source code
./user-ebuild /path/to/category/package/package-version.ebuild unpack

# Compile the package
./user-ebuild /path/to/category/package/package-version.ebuild compile

# Create a binary package (.xpak or .gpkg.tar)
./user-ebuild /path/to/category/package/package-version.ebuild package

# Install to a custom temporary directory (Merge)
ROOT=/tmp/my-install ./user-ebuild /path/to/category/package/package-version.ebuild merge

# Update Manifest (for local overlays)
./user-ebuild /path/to/category/package/package-version.ebuild manifest
```

## How It Works

1. **Sandbox Setup**: It creates a sandbox root at `/tmp/gentoo-user-ebuild-${USER}`.
2. **Namespace Isolation**: Uses `bwrap --unshare-all` to isolate IPC, PID, and Mount namespaces.
3. **Bind Mounts**:
   - `/` (Host) -> `/` (Sandbox, Read-Only)
   - `/tmp/gentoo-user-ebuild-${USER}/var/tmp/portage` -> `/var/tmp/portage` (Writable)
   - `/tmp/gentoo-user-ebuild-${USER}/var/cache/distfiles` -> `/var/cache/distfiles` (Writable)
4. **Environment**: It sets `FEATURES="-userpriv -usersandbox -sandbox -userfetch"` to disable Portage's internal privilege dropping and sandboxing, which are redundant or incompatible with the `bwrap` environment.

## Sandbox Directory Structure

The script uses `/tmp/gentoo-user-ebuild-${USER}/` as the base. Inside, you will find:
- `var/tmp/portage/`: Build working directory.
- `var/cache/distfiles/`: Downloaded source archives.
- `var/cache/binpkgs/`: Generated binary packages.
- `etc/portage/`: A copy of your system's Portage configuration.

## Cleanup

Since the sandbox is located in `/tmp`, it is typically cleared on reboot. To manually clean up:
```bash
rm -rf /tmp/gentoo-user-ebuild-${USER}
```

## Limitations

- **System Installation**: This script is designed for building and testing. It cannot install packages into your live system's `/usr` or `/lib` because the root filesystem is mounted read-only.
- **Dependencies**: It relies on your host system's installed libraries and headers to satisfy build dependencies. If a package requires a dependency not present on the host, the build will fail.
