# Codex 0.166.0 / Android 17 ARM64 / NDK r30

This experiment branch is based on upstream OpenAI Codex release
[`rust-v0.166.0`](https://github.com/openai/codex/releases/tag/rust-v0.166.0)
(commit `6ff1e1ca3b30fc8b914720681fcf5fe62e95f856`). All Android-specific build fixes are included
in one child commit; upstream and the fork's `main` branch are unchanged.

## Target

- Exactly one GitHub Actions job on `ubuntu-24.04`, triggered by pushes to
  `ci/android17-v0.166.0-arm64` or `workflow_dispatch`; no build matrix.
- `aarch64-linux-android`, Android API 37 (Android 17).
- Android NDK r30 (`30.0.16248370`).
- Rust `1.98.0` **for this workflow only**. The upstream workspace retains
  its own `rust-toolchain.toml` at 1.95.0 for other targets.
- OpenSSL `3.5.9` built statically for Android arm64.
- V8 compiled from source (`V8_FROM_SOURCE=1`).
- `mxc-sdk` restricted to Windows dependencies (still a dev dependency for
  portable tests), avoiding attempts to compile the Windows-only SDK for Android.

## Why Rust 1.98?

Rust 1.95's `std::fs::File::lock()` / `try_lock()` APIs are not implemented
for Android, including in the local ChatGPT credential store. Login can fail with
`persist_failed: lock() not supported`. Rust 1.98 implements the standard
file-locking APIs on Android via `flock(2)`:
https://github.com/rust-lang/rust/pull/157038.
This fixes all ordinary `File` lock calls without replacing individual call
sites or weakening cross-process exclusion.

## Static C++ runtime

Cargo links the final `codex` executable using NDK `clang++` with
`-static-libstdc++` and `-lc++_static`. The library name in the flag is
historical; Android's implementation is LLVM libc++. This follows the
[Android NDK C++ runtime guidance](https://developer.android.com/ndk/guides/cpp-support).

The packaging step checks ELF `DT_NEEDED` and fails if the binary still
requires `libc++_shared.so`. The published archive contains only:

- `codex` (Android ARM64 executable)
- `SHA256SUMS`

System Android/Bionic libraries may remain dynamically linked. No APK is built.

## Validate on Termux / Android

After downloading the new artifact:

```sh
tar -xzf codex-android17-aarch64.tar.gz
sha256sum -c SHA256SUMS
./codex --version
./codex
```

Then complete a ChatGPT login and run a real conversation. CI cross-compilation
and ELF checks do **not** constitute a runtime authentication test.

## Lockfile on the Android CI runner

The upstream `Cargo.lock` is retained unmodified in this release-based branch.
On this cross-target build, Cargo 1.98 needs to reconcile the lockfile with the
Android-specific dependency cfg adjustment. The Android-only build deliberately
omits `--locked`; Cargo may update the CI worktree copy of `Cargo.lock` while
respecting pinned git sources and existing locked package versions where valid.
This does not modify the repository's committed lockfile.
