# secureboot-nobara.sh
 shell script to enable secure boot in nobara
## Notes
 The shell script should work for other distributions, as long as you swap line 13 and 14 with your distribution's sbctl package. You can find the exact command from the ## Install part in https://github.com/Foxboron/sbctl
 
## Pre-use steps
 1. Enter your bios, and reset secure boot to setup mode
 2. **DO NOT** enter any other operating system after this. head straight into the nobara boot you want to sign for secure boot
## Usage
Clone and cd into this repository with 
```bash
git clone https://github.com/degenerate-kun-69/nobara-secure-boot.git/ && cd nobara-secure-boot
```
then run the script as root with 

```bash 
sudo sh secureboot-nobara.sh
```
## Issues
Please open an issue in [#issues](https://github.com/degenerate-kun-69/nobara-secure-boot/issues) so i can find the fixes

## Contributions
Everyone is welcome to contribute. fork into this repository and open a pull request along with the description of any changes

---

# sign-secureboot-nobara

Small helper script for signing Nobara/Fedora Secure Boot artifacts with already enrolled `sbctl` keys.

This script is made for the normal recurring case when keys are already present and enrolled, and you only need to sign new kernels, EFI binaries, or external kernel modules after updates.

## What it does

Depending on mode, the script can:

- enroll keys one time
- sign all installed kernel images
- sign modules for all installed kernels
- sign only the currently booted kernel and its modules
- validate signatures without changing anything
- sign unsigned EFI binaries reported by `sbctl verify`

Kernel modules are signed with the kernel `sign-file` helper. EFI binaries and kernel images are signed with `sbctl`.

## Requirements

You need:

- Nobara or Fedora-like system
- root privileges
- `sbctl`
- `modinfo`
- `depmod`
- `xz` for `.ko.xz` modules
- `zstd` for `.ko.zst` modules

The script can install `sbctl` automatically unless you use `--no-install`.

## Installation

Install the script into `/usr/local/sbin` so it can be called from anywhere:

```bash
sudo install -m 0755 ./sign-secureboot-nobara /usr/local/sbin/sign-secureboot-nobara
```

Check that shell can find it:

```bash
sudo which sign-secureboot-nobara
```

Expected output:

```text
/usr/local/sbin/sign-secureboot-nobara
```

After that you can run it simply as:

```bash
sudo sign-secureboot-nobara --validate-only
```

## One-time initial setup

If you do not have your Secure Boot keys created and enrolled yet:

```bash
sudo sign-secureboot-nobara --enroll
```

If you need Microsoft keys too, for example for Windows dual boot:

```bash
sudo sign-secureboot-nobara --enroll --microsoft
```

This is usually needed only once.

## Typical usage

Validate everything without modifying files:

```bash
sudo sign-secureboot-nobara --validate-only
```

Sign all installed kernels, modules, and EFI binaries:

```bash
sudo sign-secureboot-nobara --sign-all
```

Sign only current booted kernel and its modules:

```bash
sudo sign-secureboot-nobara --sign-current
```

Sign all kernels and modules, but skip EFI binaries:

```bash
sudo sign-secureboot-nobara --sign-all --no-efi
```

Validate only modules:

```bash
sudo sign-secureboot-nobara --validate-only --no-efi --no-kernels
```

## Recommended practical workflow

For normal maintenance, these two commands are enough:

```bash
sudo sign-secureboot-nobara --validate-only
sudo sign-secureboot-nobara --sign-all
```

If you want faster manual run and you only care about current booted kernel:

```bash
sudo sign-secureboot-nobara --sign-current
```

This is faster, but it does not sign fallback kernels or not-yet-booted newly installed kernels.

## DNF hook

If you want automatic signing after kernel or module related package updates, create DNF post-transaction action.

Create directory if needed:

```bash
sudo mkdir -p /etc/dnf/plugins/post-transaction-actions.d
```

Create file:

```bash
sudoedit /etc/dnf/plugins/post-transaction-actions.d/sign-secureboot-nobara.action
```

Put this content there:

```text
kernel-core*:any:/usr/local/sbin/sign-secureboot-nobara --sign-all --no-efi
kernel-modules*:any:/usr/local/sbin/sign-secureboot-nobara --sign-all --no-efi
kernel*:any:/usr/local/sbin/sign-secureboot-nobara --sign-all --no-efi
akmod-*:any:/usr/local/sbin/sign-secureboot-nobara --sign-all --no-efi
*kmod*:any:/usr/local/sbin/sign-secureboot-nobara --sign-all --no-efi
nvidia*:any:/usr/local/sbin/sign-secureboot-nobara --sign-all --no-efi
```

This is safer than `--sign-current` for unattended updates, because it covers all installed kernel versions.

## Modes

### `--enroll`

One-time setup mode. It creates keys if missing, enrolls them, then signs all artifacts.

### `--sign-all`

Signs:

- EFI binaries reported by `sbctl verify`
- all `/boot/vmlinuz-*`
- all modules under `/lib/modules/*`

This is the safest recurring mode.

### `--sign-current`

Signs:

- EFI binaries reported by `sbctl verify`
- `/boot/vmlinuz-$(uname -r)`
- modules under `/lib/modules/$(uname -r)`

This is faster, but narrower in scope.

### `--validate-only`

Does not modify anything. It only checks current signing state and exits with non-zero status if unsigned items are found.

This mode is useful for hooks, testing, and troubleshooting.

## Notes

- The script assumes `sbctl` key material is in:
  - `/var/lib/sbctl/keys/db/db.key`
  - `/var/lib/sbctl/keys/db/db.pem`
- Module signing uses the matching kernel `sign-file` helper from:
  - `/lib/modules/<kernel>/build/scripts/sign-file`
  - or `/usr/src/kernels/<kernel>/scripts/sign-file`
- Compressed modules are decompressed, signed, and compressed again.
- `depmod` is executed after module signing.

## Exit behavior

The script exits with non-zero status on validation failure or signing errors. This is intentional, so package hooks can detect problems.

## Typical troubleshooting

Check what `sbctl` sees:

```bash
sbctl status
sbctl verify
```

Check signer of one module:

```bash
modinfo -F signer /path/to/module.ko
```

For compressed module, unpack it temporarily first.

If script says key material is missing, but you are sure keys were already enrolled, verify that `sbctl` is using the expected key location.

## Why `--sign-all` is usually better than `--sign-current`

`--sign-current` is useful when you want a quick local fix for the running system.

But it leaves behind:

- other installed kernels
- fallback kernels
- newly installed kernels which are not booted yet
- modules for those other kernel versions

Because of this, `--sign-all` is the better default for package hooks and for regular maintenance.
