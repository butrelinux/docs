# butrelinux documentation

These documents describe the implementation in this repository, not just the user-facing feature list.

- [Architecture and image composition](architecture.md) — how recipes, shared modules, filesystem overlays, and custom BlueBuild modules combine.
- [Kernel variants and custom-kernel internals](kernels.md) — kernel selection, NVIDIA integration, initramfs, signing, and risk areas.
- [Build and test workflows](development.md) — CI pipeline, local QEMU testing, ISO assembly, and contributor workflow.
- [Operations and security](operations.md) — upgrades, signature verification, package-management boundaries, Nix persistence, and troubleshooting.
- [Repository map](repository-map.md) — where to make common changes and how the files relate.

## Scope and caveats

This documentation was written against the repository source. Where the code itself labels behavior untested or where workflows appear inconsistent, that is called out rather than treated as a supported guarantee. External dependencies (BlueBuild, CentOS Stream repositories, COPR, ELRepo, NVIDIA's installer, and image-builder) can change independently of this repository.
