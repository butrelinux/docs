# Development, builds, and tests

## CI overview

There are three main automation paths and one retry helper.

### Container image builds — `.github/workflows/build.yml`

- Runs weekly (Sunday at 00:00 UTC) or manually through `workflow_dispatch`.
- Uses a seven-entry matrix corresponding to the seven recipes.
- Calls `blue-build/github-action@v1.13` with each recipe, the Cosign signing secret, GHCR token, build-space optimization, Chunkah, and install verification.
- Requests `contents: read`, `packages: write`, and `id-token: write` permissions.

This is the primary path that turns recipe YAML and local modules into published container images. A recipe build failure can be caused by source changes here or by external repositories and upstream packaging.

### Image smoke tests — `.github/workflows/image-tests.yml`

Runs after the `bluebuild` workflow completes successfully, or manually. For each of the seven variants it:

1. Pulls the published image and `ghcr.io/osbuild/image-builder-cli:latest`.
2. Generates an ephemeral guest password and a temporary Image Builder TOML blueprint.
3. Builds a QCOW2 disk from the bootc image.
4. Boots QEMU (KVM if available, otherwise TCG emulation), forwarding host port 2222 to guest SSH.
5. Waits for SSH and collects OS release, kernel, service state, image metadata, Plasma version, and NVIDIA/Mesa details.
6. Runs `.github/tests/test.sh`, records a Markdown report, and uploads per-variant results.
7. Aggregates test output into a combined README/report and commits the generated README to the default branch.

The tests are integration smoke tests, not exhaustive desktop, hardware, upgrade, or security tests. A VM boot and successful SSH prove that a basic boot path works; they do not prove that a desktop session starts on a real GPU or that every peripheral is supported.

### ISO build and release — `.github/workflows/build-iso.yml`

Runs monthly (the first day of the month at 00:00 UTC) or manually. The workflow downloads CentOS Stream 10's `images/boot.iso` and `images/efiboot.img`, extracts the ISO tree, generates a kickstart for every entry in its `IMAGES` variable, injects a generated GRUB menu into ISO/EFI paths, and rebuilds a hybrid bootable ISO. It verifies the resulting ISO and uploads an artifact. Subsequent jobs attempt QEMU boot tests and then create a dated GitHub Release.

`kickstarts/base.cfg` is a template: `@IMAGE@` is replaced with a variant name. The generated kickstart installs the corresponding `ghcr.io/butrelinux/<variant>:latest` ostree container and uses `bootc switch --mutate-in-place --enforce-container-sigpolicy` in `%post`.

**Known source-level mismatch:** the ISO build's `IMAGES` list currently contains `lts lto lto-nvidia ml ml-nvidia hyperscale hyperscale-nvidia`, while the later ISO `boot-test` matrix names `hwe nvidia lto-hwe lto-nvidia`. Those test names do not match the generated kickstart names and the current seven-image matrix. Before relying on the ISO workflow as a release gate, reconcile the matrix with the actual generated variants and verify that each requested `*.cfg` exists in the ISO.  This is a legacy thing from the old repository when bluefin was the upstream.  we'll deal with it later.

### Automatic retry — `.github/workflows/auto-retry.yml`

Watches completed runs named `bluebuild`. If a run fails and has fewer than three attempts, it waits 90 seconds and asks GitHub CLI to rerun failed jobs. This is useful for transient registry/repository failures, but repeated failures still need investigation; retrying is not a substitute for identifying a deterministic build problem.

## Local integration testing with QEMU

The `installation/` directory provides a local path to turn a published image into a bootable QCOW2 disk using Image Builder and launch it in QEMU.

Requirements include Bash, Podman, QEMU, KVM support (`/dev/kvm`), `qemu-img`, sudo, and network access. From `installation/`:

```bash
chmod +x go.sh
./go.sh lto:latest
```

Useful modes:

```bash
BUILDONLY=1 ./go.sh lto:latest   # build disk only
SKIPINSTALL=1 ./go.sh lto:latest # boot an existing output/test-disk.qcow2
CLEAN=1 ./go.sh lto:latest       # delete output contents, rebuild, and boot
```

`CLEAN=1` deletes the contents of `installation/output/` and cannot be combined with `SKIPINSTALL=1`. The script uses a rootful Podman storage mount and a privileged Image Builder container. Understand those privileges before running it on a machine with sensitive container workloads.

The checked-in `installation/config.toml` has a convenience user with a hard-coded password for disposable local testing. Never expose that VM to untrusted networks or reuse the password. CI does not use that password: it generates an ephemeral password and hash for each run.

## Making a change safely

1. Identify whether the change is shared or variant-specific. Shared package/configuration changes usually belong in `recipes/common/`; variant identity belongs in `recipes/<variant>.yml`.
2. For new behavior, prefer a local module under `modules/<name>/` when it needs nontrivial logic, configuration, cleanup, or validation. Add `module.yml` documentation and schema where useful.
3. Keep build-time and runtime responsibilities separate. Remove compiler/toolchain artifacts after generating deliverables, but do not remove dependencies required at runtime.
4. Run the smallest relevant validation first, then build all affected variants. Changes to common fragments affect all seven variants.
5. For boot-sensitive changes, run the QEMU path and inspect serial/SSH test output. For kernel/NVIDIA/Secure Boot changes, test representative physical hardware too.
6. Keep the recipe list, build matrix, image-test matrix, ISO variant list, and documentation synchronized.
7. Do not commit secrets, private keys, credentials, generated disks, or large build artifacts.

The project's `CONTRIBUTING.md` asks contributors to follow existing naming/style conventions, keep the repository root tidy, end files with one newline, use Bash shebangs where appropriate, and test changes before submitting them.
