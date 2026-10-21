# Building Mac.apk from source

You don't need to build anything to use Mac.apk: the signed and notarized app is on
the [Releases page](https://github.com/kksimp/Mac.apk/releases/latest). This page is
for working on the code.

## Requirements

- **macOS 26.4 (Tahoe) or newer on Apple Silicon.** This is a hard floor. The arm64
  app path needs the `com.apple.security.custom-x18-abi-toggle` entitlement, which only
  works from 26.4. Intel Macs are not supported.
- **Xcode Command Line Tools** (`xcode-select --install`): clang, swiftc, codesign.
- **Homebrew**, then `brew install cmake openjdk@21 capstone apktool`.
  - `apktool` is only used once, on the first build, for the `aapt2` that builds the
    Android platform resource file from AOSP source.
  - `brew install libpng libvpx` is optional (PNG decode override, VP8/VP9 video). The
    build warns and carries on without them.
- **Xcode 26** (the full app) and its Metal toolchain only if you rebuild ANGLE from
  source. The compiled ANGLE and MoltenVK libraries are committed in
  `thirdparty/angle-prebuilt/`, so most people never need to.

`scripts/check-requirements.sh` checks all of this and prints the install command for
anything missing. `scripts/check-requirements.sh --install` installs it for you.

## Build and run

```bash
scripts/check-requirements.sh
scripts/rebuild_and_run.sh
```

`rebuild_and_run.sh` builds the runtime, the Java framework and the launcher, installs
the result as `/Applications/Mac.apk.app` (override with `MACAPK_CANONICAL_APP`), and
opens it. Pass `--no-promote` to build without touching your installed app.

Step by step instead:

```bash
runtime/scripts/build.sh                # runtime, framework jars, translator -> runtime/build/
scripts/build_launcher.sh               # the app bundle -> build/Mac.apk.app
debug/build_java_only.sh                # fast check that the Java side compiles
```

Run an APK straight from the build, without installing it:

```bash
runtime/build/bin/macapk-runtime --apk /path/to/app.apk --orientation landscape
```

Logs go to stderr. For an installed app, its log is in `~/Library/Logs/macapk/`.

## Optional: ANGLE from source

```bash
xcodebuild -downloadComponent MetalToolchain
scripts/build_angle.sh
```

That pulls about 25 GB and takes 30 to 60 minutes the first time.

## Building on GitHub Actions

The build runs on GitHub's `macos-26` runner: Apple Silicon, macOS 26.6.2,
Xcode 26.6 with the macOS 26.5 SDK. That is the project's target platform --
the arm64 guest path needs the macOS 26.4+ kernel and the build needs the
macOS 26 SDK (`PHAssetResource.contentType` is `API_AVAILABLE(macos(26.0))`),
so nothing is emulated and no local Mac is required.

| Workflow | Trigger | What it does |
|---|---|---|
| `.github/workflows/ci.yml` | every push to `main`, every pull request | Builds the runtime, the app and the `.dmg`, verifies the result, uploads the artifacts |
| `.github/workflows/release.yml` | a `v<VERSION>` tag | The same build with the release version baked in, then publishes the `.dmg` to the GitHub Release |

Both call the same composite action, `.github/actions/build-macapk/action.yml`,
so there is exactly one definition of how the project is built. It runs the
steps above in the documented order and then checks the result: the runtime
binary loads (`--help`), the bundle is self-contained
(`scripts/audit_bundle_deps.sh`), every required entitlement is present
(`debug/entitlements_check.sh`) and the signature verifies.

The release workflow signs and notarizes when the repository secrets
`MACAPK_CERT_P12`, `MACAPK_CERT_PASSWORD`, `MACAPK_SIGN_IDENTITY`,
`MACAPK_NOTARY_KEY`, `MACAPK_NOTARY_KEY_ID` and `MACAPK_NOTARY_ISSUER` are
set. Without them the build is ad-hoc signed, and the workflow creates the
release as a **draft**: a downloaded ad-hoc signed app is refused by
Gatekeeper, so publishing one publicly would hand out a file nobody can
open. With the secrets set, the release is signed, notarized, stapled and
published for real.

## Building on a Mac without Xcode 26

The build needs the macOS 26 SDK (`PHAssetResource.contentType` is
`API_AVAILABLE(macos(26.0))`) and a linker that can read it. If the machine
has a full Xcode 26, `xcode-select -s /Applications/Xcode.app` is all it
needs. If only the Command Line Tools are installed and they are older than
the SDK, a matching SDK and toolchain can be unpacked without root and
pointed at with `DEVELOPER_DIR`, `SDKROOT` and `PATH`:

```bash
TC=~/Library/Developer/macapk-toolchain
export DEVELOPER_DIR=/Library/Developer/CommandLineTools   # xcrun needs a real developer dir
export SDKROOT=$TC/SDKs/MacOSX26.5.sdk
export PATH=$TC/usr/bin:$PATH
runtime/scripts/build.sh
```

## Signed builds

`scripts/build_dist.sh --dmg` makes an ad-hoc signed app and disk image.
`scripts/release.sh` makes a Developer ID signed, notarized DMG; it reads your signing
identity and notary settings from an untracked `scripts/.release.env` (start from
`scripts/.release.env.example`). Never commit that file.

## What's not in this repository

- **Test tools and the app corpus.** The project's regression gates and its collection
  of test APKs stay private. Test your change on the apps you care about, and say in
  the pull request which ones.
- **pairip data files.** Official release builds bundle a few data files used by the
  pairip compatibility code (`lib/pairip-refs`). They are not part of this repository;
  without them a source build behaves the same except for apps that need them.

## Where things are

| Path | What it is |
|---|---|
| `runtime/java/` | The Android framework reimplementation (`android.*`, `androidx.*`, ...), the boot loader and the DEX converter |
| `runtime/src/` | The native runtime: JVM host, JNI bridges, Cocoa window, ANGLE glue, APK reader |
| `runtime/translator/` | Native code: the arm64 ELF loader, the Bionic libc shim, the ARM32 to arm64 recompiler |
| `launcher/` | The SwiftUI control panel, the per-app stub and the install helper |
| `thirdparty/` | ANGLE and MoltenVK patches and prebuilt libraries, license texts |
| `scripts/` | Build, install and release scripts |
| `debug/` | Only the helpers the build itself runs, and the GLES bridge generators |

The DEX to JVM bytecode converter is also published on its own as
[dex2jvm](https://github.com/kksimp/dex2jvm).
