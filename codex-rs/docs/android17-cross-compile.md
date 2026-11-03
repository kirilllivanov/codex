# Codex 0.166.0 / Android 17 / Arm C1-Pro optimized cross-build

Branch `ci/android17-v0.166.0-c1-pro` is based *directly* on upstream stable release
[`rust-v0.166.0`](https://github.com/openai/codex/releases/tag/rust-v0.166.0)
(commit `6ff1e1ca3b30fc8b914720681fcf5fe62e95f856`), with all Android patches in **one child commit**.
Neither `main` nor the earlier successful Android branch is modified.

## Latest stable build tooling (2026-11-03)

- Target: Android 17 / API 37, `aarch64-linux-android`; runner: Ubuntu 24.04.
- Rust: **1.99.1**, latest stable; override is confined to Android CI.
  Upstream `codex-rs/rust-toolchain.toml` stays at 1.95.0.
- Android NDK: **r30 / 30.0.16248370**, currently the latest stable LTS.
- SHA-pinned GitHub Actions: checkout v7.0.1, upload-artifact v7.0.2,
  setup-android v4.0.4, dtolnay/rust-toolchain stable action with explicit
  `toolchain: 1.99.1`. All JS actions here use Node.js 24.
- OpenSSL 3.5.9 statically built; V8 built from source. Single Linux job.

## CPU tuning from HWCAP

Arm **C1-Pro**, not Qualcomm Oryon, uses **Armv9.3-A**. Rust 1.99.1
(LLVM 23) recognizes `-Ctarget-cpu=c1-pro`, but NDK r30 Clang 21 does **not**
recognize `-mcpu=c1-pro`; the initial build failed at the Clang probe.
The corrected build uses two compatible optimization paths:

- Rust: `-Ctarget-cpu=c1-pro -Ctarget-feature=-mops,-lse128` for Android
  target crates (not the x86 CI host build utilities). The CPU model optimizes
  for C1-Pro and the exclusions avoid absent HWCAP2 extensions.
- C/C++: `-O3 -march=armv9.2-a+bf16+sme2 -mtune=cortex-a720` for native target dependencies.
- OpenSSL: `-O3 -march=armv9.2-a+bf16+sme2 -mtune=cortex-a720` during its static cross-build.
  The Cortex-A720 tune is a fallback for NDK Clang 21; not exact C1-Pro tuning.
- Release `codegen-units=1` for optimization; keep `-j 2` and LTO disabled
  for GitHub runner RAM usage.

The target device reported:

```
AT_HWCAP  = 0x00000000ffffffff
AT_HWCAP2 = 0x000051a8bc9af3ff
```

The workflow checks AES, dot-product, SVE, SVE2, BF16, SME, SME2 and LRCPC3.
MOPS (HWCAP2 bit 43) and LSE128 (bit 47) are **not advertised**. Rust
explicitly disables both, and C/C++ uses a pre-v9.3 ISA base with BF16 and
SME2 enabled because this device advertises them. CPU-specific model settings
are delegated to LLVM instead of manually enabling every possible
extension. This is a **device-specific executable** and is not promised to
work on other ARM64 Android devices. The V8 GN/Ninja build configures
machine flags separately; native CFLAGS do not guarantee every V8 object
has `-mcpu=c1-pro`.

References:
- https://github.com/torvalds/linux/blob/master/arch/arm64/include/uapi/asm/hwcap.h
- https://www.arm.com/products/silicon-ip-cpu/c1-pro
- https://github.com/android/ndk/releases/tag/r30

## Android compatibility fixes

Rust >=1.98 implements Android `std::fs::File` locking using `flock`,
addressing the previous ChatGPT login `persist_failed: lock() not supported`
without hand-written wrappers:
https://github.com/rust-lang/rust/pull/157038.

The Windows-only `mxc-sdk` dependency is moved under `cfg(windows)`
(with a dev-dependency retained for tests). The upstream `Cargo.lock`
is left unchanged in Git; the Android CI build allows local Cargo
lockfile reconciliation instead of using `--locked`.

## Static C++ runtime and validation

The Android binary is linked with NDK Clang++ using
`-static-libstdc++ -lc++_static`. The packaging step checks ELF
`DT_NEEDED` and fails if `libc++_shared.so` is present. Android
system libraries like `libc.so` may remain dynamic.

The Actions artifact `codex-android17-aarch64-c1-pro-rust199-static-cxx`
contains a tarball `codex-android17-aarch64-c1-pro.tar.gz` with
only `codex` and `SHA256SUMS`. On the target device:

```sh
tar xzf codex-android17-aarch64-c1-pro.tar.gz
sha256sum -c SHA256SUMS
chmod +x codex
./codex --version
./codex
```

Complete browser OAuth login and execute a real session. CI cross-build
success alone is not proof of target instruction compatibility or speed.
