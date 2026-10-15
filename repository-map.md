# Repository map and change routing

| Path | Responsibility | Typical changes |
|---|---|---|
| `recipes/<variant>.yml` | Top-level BlueBuild recipe for one published image | Kernel choice, NVIDIA toggle, image identity |
| `recipes/common/bootstrap.yml` | Copies filesystem overlay, runs setup scripts, adjusts executable bits | Common scripts/config deployment |
| `recipes/common/dnf.yml` | Shared RPM package set and Fedora RPM addition | Desktop/system packages and firmware |
| `recipes/common/nix.yml` | Nix module settings | Nix flavor, features, build users, trusted users |
| `recipes/common/end.yml` | Fonts, service state, kernel args, repo cleanup, signing | Global finalization policy |
| `modules/custom-kernel/` | Kernel and out-of-tree module build logic | Kernel source, NVIDIA, initramfs, Secure Boot |
| `modules/nix/` | Nix installation, persistence, daemon lifecycle | Nix seeding/upgrade behavior |
| `modules/fedora-rpm/` | Install from a separate RPM release universe | Temporary repository handling and package installation |
| `modules/clean-repos/` | Repository allowlist enforcement | Which repo IDs survive into final image |
| `modules/image-info/` | Variant metadata and OS branding | Image identity fields and release metadata |
| `files/scripts/` | Build-time setup scripts invoked by recipes | SELinux policy compilation, enabling CRB/EPEL |
| `files/system/` | Files copied into the root filesystem | Shell helpers, fetch config, wallpaper, system defaults |
| `kickstarts/base.cfg` | Template for each ISO install menu entry | Anaconda installation behavior and bootc switch |
| `boot/grub.cfg` | GRUB menu template with `@MENU_ENTRIES@` insertion point | Installer menu behavior |
| `.github/workflows/build.yml` | Container image CI | Build matrix, registry publishing, signing action settings |
| `.github/workflows/image-tests.yml` | QCOW2/VM smoke tests and report aggregation | Runtime checks, SSH, test artifacts |
| `.github/workflows/build-iso.yml` | ISO generation, boot tests, release | ISO variant list, bootloader patching, release pipeline |
| `.github/workflows/auto-retry.yml` | Retry failed BlueBuild runs | Retry policy and attempt limit |
| `.github/tests/` | Remote guest test harness and blueprint template | Assertions and collected diagnostics |
| `installation/` | Local Image Builder + QEMU test helper | Reproducing boot tests manually |
| `cosign.pub` | Public key for image signature verification | Trust key rotation; keep synchronized with signing setup |

## Variant change checklist

When adding, removing, or renaming a variant, check all of these locations together:

1. Add/update `recipes/<variant>.yml`.
2. Update the `matrix.recipe` list in `.github/workflows/build.yml`.
3. Update the `matrix.variant` list in `.github/workflows/image-tests.yml`.
4. Update the `IMAGES` variable in `.github/workflows/build-iso.yml`.
5. Update the ISO boot-test matrix in the same workflow and ensure every generated kickstart exists.
6. Update the README's published image table and this documentation.
7. Check generated test/report logic for hard-coded variant lists.
