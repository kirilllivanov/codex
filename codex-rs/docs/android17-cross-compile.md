# Android 17 ARM64 CLI cross-build (experimental)

The GitHub Actions workflow at `.github/workflows/android17.yml` cross-compiles the
Rust `codex` binary for `aarch64-linux-android`. It does **not** produce an APK.

- Android target: API 37 (Android 17; SDK platform `android-37.0`)
- NDK: r30 (`30.0.16248370`)
- Rust: use `codex-rs/rust-toolchain.toml` (currently 1.95.0)
- TLS: OpenSSL 3.5.9 built statically for Android arm64 from an exact upstream commit
- V8: built from source because this repository lacks Android prebuilt archives

On a successful CI run, GitHub Actions uploads
`codex-android17-aarch64-ndk-r30` containing an archive with `codex`,
`libc++_shared.so`, and SHA256 checksums.

The Android native target is not officially supported by all workspace dependencies.
A green CI build is required before claiming Android compatibility. The build has not
yet been confirmed working, and runtime behavior on Android/Termux has not been tested.
