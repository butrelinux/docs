# Kernel variants and the custom-kernel module

## Variant matrix

| Recipe/image | Kernel module | NVIDIA module | Intended distinction |
|---|---|---:|---|
| `lts` | none (base kernel) | No | CentOS Stream 10 kernel path; the README describes this as 6.12. |
| `lto` | `cachyos-lto` | No | CachyOS LTO kernel from COPR. |
| `lto-nvidia` | `cachyos-lto` | Yes | CachyOS LTO plus NVIDIA driver modules. |
| `ml` | `ml` | No | ELRepo `kernel-ml`, intended for faster mainline updates. |
| `ml-nvidia` | `ml` | Yes | ELRepo mainline kernel plus NVIDIA. |
| `hyperscale` | `hyperscale` | No | CentOS Hyperscale SIG kernel. |
| `hyperscale-nvidia` | `hyperscale` | Yes | Hyperscale kernel plus NVIDIA. |

The recipe names and image tags are not cosmetic: they are used by CI, the `image-info` metadata, ISO kickstart generation, and published GHCR paths. Keep these references in sync when adding or renaming a variant.

## What `custom-kernel` does

The module is a substantial build-time script rather than a thin package list. The implementation:

1. Snapshots installed RPM names before adding build dependencies, so it can distinguish original packages from temporary toolchain dependencies.
2. Checks that the target is an EL10-family environment.
3. Selects a kernel source according to `kernel` (`cachyos-lto`, `ml`, `hyperscale`, or `stock`).
4. Installs the required kernel/devel packages and build tools for that source.
5. Builds out-of-tree modules, including v4l2loopback and optionally NVIDIA, for the selected kernel.
6. Optionally signs kernel artifacts and creates a MOK enrollment unit when Secure Boot signing is configured.
7. Removes build trees and caches, and tries to remove packages installed only for the build.
8. Optionally generates a reproducible dracut initramfs at `/usr/lib/modules/<version>/initramfs.img`.
9. Checks that exactly one directory containing a `vmlinuz` remains under `/usr/lib/modules`, and verifies NVIDIA module files when NVIDIA is enabled.

Build dependency cleanup is intentionally cautious: it records packages that were not present at the start and avoids removing packages still required by packages outside the tracked removal set. This logic is safety-sensitive; changes should be tested in a clean image build.

## Configuration knobs

The current recipes set `initramfs: true` for non-stock kernel variants and set `nvidia` explicitly to true or false. The module documentation also describes optional `sign` configuration with a private key, certificate, and MOK password supplied through BlueBuild secrets. 

## NVIDIA-specific behavior

When enabled, the module installs the NVIDIA driver from an upstream `.run` payload and checks for the expected kernel modules (`nvidia`, `nvidia-drm`, `nvidia-modeset`, `nvidia-peermem`, and `nvidia-uvm`). It also installs NVIDIA Container Toolkit support and writes a CDI-generation systemd service. This means a successful image build verifies that artifacts exist, but does not prove that the driver works with every physical GPU, firmware setting, display manager, or Secure Boot configuration.

The image smoke test attempts to read an NVIDIA driver version only for variants whose image metadata mentions NVIDIA. Hardware validation is still advisable before relying on an upgrade.

## Secure Boot considerations

The module has two concepts that should not be conflated:

- **Kernel image signing**: whether the kernel image itself is signed by the configured key or retained with its vendor signature.
- **Kernel module signing**: signing out-of-tree modules so the kernel can load them under Secure Boot policy.

For custom kernels, signing behavior depends on the source kernel and the supplied `sign` configuration. The module can generate an enrollment unit for a Machine Owner Key, but the machine owner must still complete the enrollment process in firmware/MOK tooling. Verify the actual boot path on the target machine; an image build alone is not proof of Secure Boot compatibility.

## Kernel-change checklist

- Update the appropriate recipe's `kernel`, `initramfs`, and `nvidia` values.
- Check repository URLs, package names, and release-version assumptions in `custom-kernel.sh`.
- Confirm the resulting image contains exactly one intended kernel under `/usr/lib/modules`.
- Verify module ABI alignment: `kernel-devel`, NVIDIA, and v4l2loopback must match the installed kernel.
- Build all affected variants and inspect logs for dependency cleanup failures.
- Boot-test with the matching ISO/VM path and, where relevant, real hardware and Secure Boot enabled.
- Update this document and the user-facing variant list if behavior or support status changes.
