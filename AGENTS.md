# Repository Guidelines

## Project Overview

This repository builds an unofficial **LineageOS 23.2 / Android 16** ROM for the Motorola Moto G8 Plus (`doha`, product `amogus_doha`, Snapdragon 665).

It is a thin ROM port and build-overlay repository, not an application source tree:

- `default.xml` pins the upstream Android/LineageOS projects.
- `patches/` contains device, kernel, SELinux, HAL, and compatibility fixes.
- `build.sh` synchronizes upstream sources, applies patches, builds the ROM, and copies flashable artifacts.

Keep changes device-specific and narrow. This port pairs Android 16 with a legacy 4.14 kernel, so preserve the rationale in `README.md` when changing compatibility behavior.

## Architecture & Data Flow

The build pipeline is:

```text
default.xml
  -> repo init/repo sync into an external source tree
  -> discover patches/<upstream-project-path>.patch
  -> reset/reapply patches idempotently
  -> create device/motorola/amogus_doha/kernel-headers sentinel file
  -> source build/envsetup.sh
  -> breakfast amogus_doha userdebug
  -> mka target-files-package bacon
  -> copy ROM ZIP, boot/dtbo/vbmeta images, and SHA-256 sums
```

Important properties of this flow:

- Upstream source lives in a separate Git checkout, by default the git-ignored `./lineage/` directory inside this working tree or the directory passed to `build.sh`; it is not tracked by this repository.
- A patch owns all local changes to one upstream project. Its path must exactly mirror that project's repository path. Example: `patches/device/motorola/common.patch` targets `device/motorola/common`.
- `build.sh` resets and cleans patched projects before applying their patch. Manual edits in the synchronized tree are not durable.
- `.repo/doha-patches` in the source tree records rendered-patch and worktree hashes so unchanged trees retain timestamps and incremental-build state.
- Runtime behavior is primarily declarative: Android makefiles, `Android.bp`, init `.rc` files, VINTF manifests, SELinux policy, and narrowly scoped C/C++/Java patches. There is no application-level object graph or state store.
- Configuration follows Android ownership boundaries: `PRODUCT_*` and `BOARD_*` variables for product/board configuration, init/VINTF for services, and SELinux files for labels.

## Key Directories

### Local repository paths

| Path | Purpose |
| --- | --- |
| `patches/device/motorola/` | Product, board, module-loading, fingerprint, power-HAL, and VINTF changes. |
| `patches/device/qcom/` | Qualcomm common and legacy-UM SELinux corrections. |
| `patches/device/lineage/` | Removes the duplicate `sched_energy_aware` genfscon shared with the Qualcomm policy trees. |
| `patches/vendor/qcom/` | Qualcomm common GPS/QCC packaging fixes, plus audio, thermal HAL, and HIDL interface patches under `opensource/`. |
| `patches/vendor/motorola/` | Motorola proprietary blob packaging, including `libqsap_sdk.so`. |
| `patches/kernel/motorola/` | Kernel 4.14 backports and fixes. |
| `patches/packages/modules/` | Connectivity/netd/BPF compatibility with kernel 4.14. |
| `patches/hardware/` | Motorola `libqsap_shim` build disable, and NXP NFC HAL retry delay. |
| `patches/system/` | Optional inherited `system/core` SafetyNet/Play Integrity patch. |

### Upstream paths referenced by the manifest

These are synchronized project roots, not local source directories:

- `device/motorola/amogus`, `device/motorola/amogus_doha`, `device/motorola/common`, `device/motorola/targets`: device/product configuration.
- `device/motorola/amogus-kernel`: kernel packaging and exported headers.
- `device/qcom/common`, `device/qcom/sepolicy_vndr/legacy-um`: Qualcomm configuration and policies.
- `kernel/motorola/msm-4.14`: Motorola kernel fork.
- `packages/modules/Connectivity`: netd and BPF networking.
- `vendor/motorola/amogus`, `vendor/motorola/amogus_doha`: proprietary vendor content.
- `vendor/qcom/opensource/{thermal,audio/sm8150,interfaces}`: Qualcomm HALs and interfaces.

## Development Commands

### Prerequisites and directories

```sh
git clone https://github.com/christiaan/lineage-23-amogus_doha.git doha-lineage
mkdir -p ~/android/doha-23.2 ~/android/ccache/doha ~/doha/zips
```

### Docker build (recommended)

Run from the directory containing the `doha-lineage` clone:

```sh
docker run --rm -it \
  --user "$(id -u):$(id -g)" -e HOME=/tmp \
  -v "$PWD/doha-lineage:/manifest:ro" \
  -v ~/android/doha-23.2:/src \
  -v ~/android/ccache/doha:/ccache -e CCACHE_DIR=/ccache \
  -v ~/doha/zips:/zips -e ZIP_DIR=/zips \
  --entrypoint /manifest/build.sh \
  lineageos4microg/docker-lineage-cicd:latest /src
```

### Native build

```sh
./build.sh ~/android/doha-23.2
```

The default source directory is `./lineage`.

### Useful build options

```sh
JOBS=16 ./build.sh ~/android/doha-23.2
SYNC_JOBS=2 ./build.sh ~/android/doha-23.2
SKIP_SYNC=1 ./build.sh ~/android/doha-23.2
ZIP_DIR=~/doha/zips SKIN_THROTTLE_C=40 ./build.sh ~/android/doha-23.2
```

- `JOBS`: build parallelism; defaults to `nproc`.
- `SYNC_JOBS`: sync parallelism; defaults to `4`. Keep it low because Google Git servers return HTTP 429 when overloaded.
- `SKIP_SYNC=1`: reuse an existing synchronized tree.
- `CCACHE_SIZE`: ccache limit; defaults to `50G`.
- `ZIP_DIR`: artifact destination; defaults to `zips/` in the source directory's parent.
- `SKIN_THROTTLE_C`: thermal skin threshold in whole degrees Celsius; defaults to `40`.
- `MANIFEST_URL` / `MANIFEST_BRANCH`: override the manifest repository and branch.

### Flash/install smoke checks

```sh
fastboot flash boot lineage-23.2-*-boot.img
adb sideload lineage-23.2-*.zip
```

For an in-place update:

```sh
adb reboot sideload-auto-reboot
adb wait-for-sideload
adb sideload lineage-23.2-*.zip
```

Use the copied artifacts under `ZIP_DIR`, not dated ZIPs under `out/`: `bacon` hard-links its output and later builds overwrite the linked contents.

There are no repository-defined lint, format, type-check, unit-test, or package-install commands.

## Code Conventions & Common Patterns

### Shell orchestration

- `build.sh` uses Bash with `set -euo pipefail`; keep strict mode enabled except around AOSP shell entry points that are not `errexit`/`nounset` clean. Check those commands' statuses explicitly.
- Use lowercase shell functions and uppercase environment variables. Quote expansions and keep commands POSIX-compatible unless Bash is required.
- Validate externally supplied numeric values before starting expensive work.
- Do not write Git identity or safe-directory settings to `~/.gitconfig`; the script injects them through `GIT_CONFIG_*` environment variables.

### Patch organization

- Store exactly one Git diff per upstream project at `patches/<project-path>.patch`.
- Generate diffs relative to the upstream project root, including the correct `a/` and `b/` paths.
- A project patch may contain only that project's changes. Update `default.xml` separately when changing an upstream pin.
- Preserve explanatory comments for non-obvious compatibility work. `README.md` explains why each important patch exists.
- When changing an existing patch, regenerate the complete project-relative diff from the pinned upstream state so the patch applies cleanly in one pass.

### Android and upstream code

- Follow existing upstream style and Android naming: `PRODUCT_*`, `BOARD_*`, `LOCAL_*`, `Android.bp`, `hidl_interface`, and init `.rc` conventions.
- Use Android logging (`ALOGW`, `ALOGE`, `Log.e`) and explicit null checks at HAL boundaries.
- Keep kernel-version fallbacks narrowly scoped. Existing warnings/null-safe fallbacks for BPF on 4.14 prevent boot loops; do not turn them back into fatal errors without device-level validation.
- Services, threading, and asynchronous behavior belong to Android HAL/init/kernel code. There is no repository-specific async framework, dependency-injection mechanism, or global state-management pattern.
- Runtime state is distributed across Android properties, init service state, `/proc`, `/sys`, `/mnt/vendor/persist`, and BPF maps. Choose the existing subsystem that owns the state instead of introducing another global store.
- The `@SKIN_THROTTLE_MC@` placeholder is rendered from `SKIN_THROTTLE_C` as millidegrees Celsius by `render_patch()`.
- Preserve generated-file expectations. Vendor makefiles and HIDL files may be generated upstream; if they must change, patch the generated result in the correct form rather than bypassing the build system.

### Manifest and source-tree rules

- Keep project revisions pinned. Change a revision only as part of a deliberate upstream update.
- When building from this checkout (`MANIFEST_URL` unset), commit `default.xml` before `build.sh` runs: `repo` reads the committed branch, not the working tree, and the script refuses otherwise. A detached checkout also requires `MANIFEST_BRANCH`.
- Keep `device/motorola/amogus_doha/kernel-headers` an empty regular file in the synchronized tree. Do not replace it with a directory; AOSP automatically adds that path to the include path.
- Do not restore the removed `hardware/qcom/bootctrl` Soong namespace import, incompatible `qccsyshal@1.2` service, source-built `libqsap_sdk`, or strict BPF kernel-version failures without revalidating Android 16 compatibility.

## Important Files

| File | Role |
| --- | --- |
| `build.sh` | Main entry point: source sync, patch lifecycle, ROM build, artifact packaging. |
| `default.xml` | Google `repo` manifest: remotes, default Lineage revision, and all upstream project pins. |
| `README.md` | Authoritative setup, build, install, hardware status, and patch-rationale documentation. |
| `.gitignore` | Ignores only the default `/lineage/` source tree. |
| `patches/device/motorola/targets.patch` | Selects the curated vendor kernel-header export. |
| `patches/device/motorola/amogus.patch` | Product makefile, proprietary-file list, SELinux labels, and VINTF behavior. |
| `patches/device/motorola/amogus_doha.patch` | Board-specific kernel-module loading. |
| `patches/device/motorola/common.patch` | Power HAL, fingerprint init/labels, and Qualcomm VINTF changes. |
| `patches/packages/modules/Connectivity.patch` | BPF/netd kernel-version fallbacks and null-safe map handling. |
| `patches/kernel/motorola/msm-4.14.patch` | Kernel WPA3 and syscall/backport fixes. |
| `patches/vendor/qcom/opensource/thermal.patch` | Thermal HAL configuration and init service changes. |
| `patches/vendor/qcom/opensource/interfaces.patch` | Added vendor FM HIDL interface. |
| `patches/system/core.patch` | Optional inherited SafetyNet/Play Integrity patch; remove it if unwanted. |

## Runtime/Tooling Preferences

- **Host:** x86-64 Linux. Docker is the documented, lowest-friction environment; native builds require the LineageOS build dependencies and Google `repo`.
- **Container:** `lineageos4microg/docker-lineage-cicd:latest` supplies Bash, Google `repo`, Git LFS, and the Android build toolchain.
- **Git identity:** `build.sh` injects `safe.directory` and a fallback user identity through `GIT_CONFIG_*` when they are not already configured; it does not modify `~/.gitconfig`.
- **Source manager:** Google `repo`, not npm/pip/cargo. There is no package-manager manifest or lockfile.
- **Shell/runtime:** Bash; the Android build itself runs through `build/envsetup.sh`, Soong, and Make.
- **Versions:** LineageOS 23.2, Android 16, AOSP tag `android-16.0.0_r4`; the device kernel is based on 4.14 and reports `4.14.357-openela+`.
- **Resources:** about 350 GB free disk, 32 GB RAM, and at least 32 GB swap. ccache can add up to 50 GB.
- **Build product:** `amogus_doha`, `userdebug`, unsigned with AOSP test keys, without GApps.
- **Build outputs:** `lineage-<LINEAGE_VERSION>.zip`, matching `-{boot,dtbo,vbmeta}.img` files, and `.sha256sum` under `ZIP_DIR`; `LINEAGE_VERSION` is typically `23.2-<date>-UNOFFICIAL-amogus_doha`.

## Testing & QA

There is no repository-local test framework, test directory, coverage configuration, CI workflow, or documented `atest`/CTS/VTS command. CTS, VTS, Tradefed, and upstream test projects are present in the manifest, but this repository provides no documented command or CI job to run them.

Primary QA is:

1. **Build:** complete `mka target-files-package bacon` build after `breakfast`. Android build-time validation includes SELinux `checkpolicy`, `*_contexts`/`sepolicy_test`, and `checkvintf`; init `.rc` files that reference undeclared HIDL interfaces also fail the build through `host_init_verifier`.
2. **Artifacts:** confirm the ZIP, boot/dtbo/vbmeta images, and SHA-256 sum were produced under `ZIP_DIR`.
3. **Hardware:** flash on a Moto G8 Plus and exercise the affected feature.

The README records working boot, SIM/data, Wi-Fi/WPA3, Bluetooth HID/A2DP, NFC tag reading, GPS, FPC fingerprint, vibration, camera, audio, charging, VoLTE, hotspot, and USB tethering. Still unverified: Bluetooth HFP call audio, tethered traffic over cellular data, and 464XLAT on IPv6-only networks. FocalTech/Novatek touch support is also untested on the corresponding panels.

For runtime changes, perform the narrowest relevant device check possible and update the status/rationale documentation. A successful compilation does not establish hardware behavior.
