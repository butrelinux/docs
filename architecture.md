# Architecture: how an image is assembled

## 1. The image model

butrelinux is built as a bootc container image based on `quay.io/centos-bootc/centos-bootc` at image version `c10s`. BlueBuild consumes one recipe per published variant. The resulting image is published to `ghcr.io/butrelinux/<variant>` and is intended to be installed and updated as an image, rather than maintained as a conventional mutable package installation.

A useful mental model is:

1. Start from the CentOS bootc base image.
2. Apply the shared operating-system configuration and desktop package set.
3. Install Nix and its runtime integration.
4. Optionally install a non-stock kernel and matching out-of-tree modules.
5. Write image identity metadata.
6. Apply final systemd, kernel-argument, repository-cleanup, and signing steps.
7. Publish the image through the BlueBuild GitHub Action.

The sequence matters: a module can rely on packages or files introduced earlier, and a final module can remove build-time dependencies only after kernel artifacts have been generated.

## 2. Recipe composition

Each top-level file in `recipes/` is a variant entry point. The recipes intentionally repeat very little common configuration:

```yaml
base-image: quay.io/centos-bootc/centos-bootc
image-version: c10s
modules:
  - from-file: common/common.yml
  # optional custom-kernel module
  - type: image-info
    source: local
    variant: <variant>
  - from-file: common/end.yml
```

`recipes/common/common.yml` composes three shared fragments in order:

- `common/bootstrap.yml`: copies the repository's `files/system/` tree into the image, runs SELinux policy and repository setup scripts, and sets executable bits on selected utilities.
- `common/nix.yml`: installs the multi-user Nix environment using the local `nix` module.
- `common/dnf.yml`: removes selected Red Hat subscription/insights packages, installs firmware/networking/KDE/audio/system utilities, installs Discover without weak dependencies, and adds `dcli-bootc` from a Fedora 44 COPR repository via the local `fedora-rpm` module.

The recipe then selects a kernel where applicable, writes variant metadata, and imports `common/end.yml`.

### Why shared fragments exist

A change to a common fragment affects every recipe that imports it. This is the right place for a package or policy that should be invariant across variants. Kernel choice, NVIDIA support, and the `image-info` variant string belong in each recipe because they define the variant's identity. When adding a new variant, ensure the recipe, build matrix, image-test matrix, ISO `IMAGES` list, and any ISO boot-test assumptions remain aligned.

## 3. Finalization (`common/end.yml`)

The final shared fragment currently:

- installs JetBrains Mono, Nerd Fonts Symbols Only, and Noto font families;
- enables SDDM, tuned, tuned-ppd, earlyoom, and NetworkManager;
- disables `nfs-client.target` and masks `rpcbind`, `rpcbind.socket`, and `system-flatpak-setup.timer`;
- adds kernel arguments `amdgpu.ppfeaturemask=0xffffffff` and `split_lock_detect=off`;
- invokes the local `clean-repos` module;
- invokes BlueBuild's signing module.

These settings are global defaults, not per-kernel options. In particular, the AMDGPU feature mask and split-lock setting apply to all recipes that import this fragment. Changes should be deliberate and validated on hardware where possible.

## 4. Filesystem overlay

`recipes/common/bootstrap.yml` copies `files/system` to `/` in the image. The source directory mirrors the destination hierarchy:

- `files/system/etc/profile.d/10-butrelinux-alias.sh` defines shell aliases and the `pkg-add`, `pkg-rm`, and `pkg-run` wrappers around Nix commands.
- `files/system/etc/butrelinux/fastfetch.json` and `files/system/usr/share/butrelinux/fastfetch.jsonc` hold Fastfetch presentation configuration.
- `files/system/usr/bin/bfetch` is the project-specific fetch wrapper.
- `files/system/usr/bin/enable-extras` provides a Distrobox-based optional application installer.
- `files/system/usr/bin/givemethekeys` toggles rpm-ostree package layering policy.
- `files/system/usr/bin/image-info.sh` prints image identity and whether the booted deployment appears signed.
- `files/system/usr/share/wallpapers/butre-default.jpg` is the default wallpaper asset.

If you add a file here, remember that it is copied into every image through the shared bootstrap. If it needs to be executable, ensure the recipe's `snippets` list or the source file mode sets that bit; the bootstrap explicitly chmods several scripts.

## 5. Custom BlueBuild modules

Local modules live under `modules/<module-name>/`. `module.yml` documents the module to BlueBuild; the implementation is usually a shell script that receives its module configuration as JSON in `$1`. The repository includes:

| Module | Role |
|---|---|
| `clean-repos` | Removes repository definitions not in a hard-coded EL10 allowlist. |
| `custom-kernel` | Selects and installs a kernel; builds optional NVIDIA/v4l2loopback modules; handles initramfs and optional Secure Boot signing. |
| `fedora-rpm` | Temporarily adds Fedora/COPR repositories and installs packages with a separate release version. |
| `image-info` | Writes `/usr/share/butrelinux/image-info.json` and patches `/usr/lib/os-release`. |
| `nix` | Installs Nix at build time and arranges persistent state, daemon services, first-boot seeding, and optional boot-time upgrades. |

`modules/fedora-rpm/fedora-rpm.tsp` defines a JSON Schema for the module's configuration. The local `source: local` references in recipes tell BlueBuild to use the repository's module implementations.

## 6. Image identity and signatures

The `image-info` module writes the variant name, vendor, tag, and expected image reference to `/usr/share/butrelinux/image-info.json`; it also changes OS-release branding. The runtime `image-info.sh` script reads that file and prefers the booted image's tag from `bootc status --json` when available. It then prints a lock emoji based on a check of `rpm-ostree status --booted` for the word `signed`.

Treat this output as a convenience indicator, not a cryptographic verification procedure. The actual verification workflow is `cosign verify --key cosign.pub <image-ref>`.
