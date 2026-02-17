# RustDesk (twekr_mod) — Local Build Guide for Windows x64

> **Base commit:** [`998b75856da4199ac009ce4135b4fba1b48099e6`](https://github.com/rustdesk/rustdesk/commit/998b75856da4199ac009ce4135b4fba1b48099e6)
>
> This guide walks you through building RustDesk **locally on Windows x64** for development.
> Every version number, commit ID, and command is pinned — follow them exactly.

---

## Table of Contents

1. [Prerequisites Overview](#1-prerequisites-overview)
2. [Install Visual Studio Build Tools](#2-install-visual-studio-build-tools)
3. [Install LLVM](#3-install-llvm)
4. [Install Rust Toolchain](#4-install-rust-toolchain)
5. [Install Python 3](#5-install-python-3)
6. [Install Git (if not already)](#6-install-git-if-not-already)
7. [Clone the Repository](#7-clone-the-repository)
8. [Install Flutter (Build Version)](#8-install-flutter-build-version)
9. [Replace Flutter Engine with RustDesk Custom Engine](#9-replace-flutter-engine-with-rustdesk-custom-engine)
10. [Patch Flutter](#10-patch-flutter)
11. [Install & Setup vcpkg](#11-install--setup-vcpkg)
12. [Install vcpkg Dependencies](#12-install-vcpkg-dependencies)
13. [Generate the Flutter–Rust Bridge](#13-generate-the-flutterrust-bridge)
14. [Build RustDesk](#14-build-rustdesk)
15. [Quick Reference — All Versions at a Glance](#15-quick-reference--all-versions-at-a-glance)

---

## 1. Prerequisites Overview

| Tool                                 | Version                                           | Why                                           |
| ------------------------------------ | ------------------------------------------------- | --------------------------------------------- |
| Visual Studio Build Tools            | 2022 (with C++ workload)                          | MSVC compiler, linker, Windows SDK            |
| LLVM + Clang                         | **15.0.6**                                        | `libclang` needed by Rust `bindgen` crate     |
| Rust                                 | **1.75**                                          | Pinned Rust toolchain for the project         |
| Python                               | 3.x (any recent)                                  | `build.py` build script                       |
| Git                                  | Any recent                                        | Cloning repos, vcpkg, submodules              |
| Flutter (for final build)            | **3.24.5**                                        | Flutter UI framework (stable channel)         |
| Flutter (for bridge generation only) | **3.22.3**                                        | Required by `flutter_rust_bridge_codegen`     |
| vcpkg                                | commit `120deac3062162151622ca4860575a33844ba10b` | C/C++ dependency manager                      |
| `cargo-expand`                       | **1.0.95**                                        | Macro expansion (bridge codegen dep)          |
| `flutter_rust_bridge_codegen`        | **1.80.1**                                        | Generates FFI bridge between Flutter and Rust |

---

## 2. Install Visual Studio Build Tools

RustDesk (and Rust on Windows in general) needs the MSVC C++ toolchain.

1. Download **Visual Studio Build Tools 2022** from <https://visualstudio.microsoft.com/visual-cpp-build-tools/>.
2. In the installer, select the **"Desktop development with C++"** workload.
    - Make sure the following components are checked (they usually are by default):
        - MSVC v143 – VS 2022 C++ x64/x86 build tools
        - Windows 10/11 SDK (latest)
        - C++ CMake tools for Windows
3. Complete the installation.

> [!NOTE]
> If you already have full Visual Studio 2022 installed with the C++ workload, you can skip this step.

---

## 3. Install LLVM

**Version: `15.0.6`**

LLVM provides `libclang`, which several Rust crates (e.g. `bindgen`) require at build time.

1. Download the LLVM **15.0.6** installer for Windows from the official GitHub releases:
   <https://github.com/llvm/llvm-project/releases/tag/llvmorg-15.0.6>

    Direct link: [`LLVM-15.0.6-win64.exe`](https://github.com/llvm/llvm-project/releases/download/llvmorg-15.0.6/LLVM-15.0.6-win64.exe)

2. Run the installer. During installation, **select the option to add LLVM to the system PATH** (for all users or current user).

3. Verify the installation by opening a **new** PowerShell window:

    ```powershell
    clang --version
    ```

    Expected output should show `clang version 15.0.6`.

4. Ensure the `LIBCLANG_PATH` environment variable is set. If it's not automatically set, add it:

    ```powershell
    # For the current session:
    $env:LIBCLANG_PATH = "C:\Program Files\LLVM\bin"

    # To set it permanently (run as Administrator, or use System Settings):
    [System.Environment]::SetEnvironmentVariable("LIBCLANG_PATH", "C:\Program Files\LLVM\bin", "User")
    ```

---

## 4. Install Rust Toolchain

**Version: `1.75`**
**Target: `x86_64-pc-windows-msvc`**

1. If you don't have Rust installed at all, install `rustup` from <https://rustup.rs/>.

2. Install the specific Rust version and set it as default:

    ```powershell
    rustup install 1.75
    rustup default 1.75-x86_64-pc-windows-msvc
    ```

3. Add the required target and component:

    ```powershell
    rustup target add x86_64-pc-windows-msvc
    rustup component add rustfmt
    ```

4. Verify:

    ```powershell
    rustc --version
    # Expected: rustc 1.75.0 (...)

    cargo --version
    ```

> [!IMPORTANT]
> Do **not** use a newer Rust version. Version `1.75` is specifically required because Rust 1.78+ introduced
> an [ABI change for `i128`](https://blog.rust-lang.org/2024/03/30/i128-layout-update.html) that breaks the
> Sciter integration used in part of the RustDesk build.

---

## 5. Install Python 3

The build script (`build.py`) requires Python 3.

1. Download and install Python 3 from <https://www.python.org/downloads/>.
2. During installation, **check "Add python.exe to PATH"**.
3. Verify:

    ```powershell
    python3 --version
    # or
    python --version
    ```

> [!TIP]
> On Windows, the `python3` command may not exist if you installed Python from python.org.
> The build script uses `python3`, so if only `python` works, you can create an alias or
> ensure `python3.exe` is in your PATH. On recent Windows installations, `python3` aliasing is
> usually handled by the Python installer or the Windows "App Execution Aliases" feature.

---

## 6. Install Git (if not already)

Download and install Git from <https://git-scm.com/download/win>.

Verify:

```powershell
git --version
```

---

## 7. Clone the Repository

```powershell
git clone <your-fork-url> rustdesk
cd rustdesk
git submodule update --init --recursive
```

> [!IMPORTANT]
> The `git submodule update --init --recursive` step is **critical**. The project depends on the
> `libs/hbb_common` submodule (hosted at `https://github.com/rustdesk/hbb_common`). The
> Flutter–Rust bridge codegen step (Step 13) will **fail** if the submodule is not initialized.

---

## 8. Install Flutter (Build Version)

**Version: `3.24.5`** (stable channel)

You will need **two different Flutter versions** during the setup process:

- **`3.24.5`** — for the actual RustDesk build (this step)
- **`3.22.3`** — for the Flutter–Rust bridge codegen step (Step 13)

We recommend using [FVM (Flutter Version Management)](https://fvm.app/) to manage multiple Flutter versions, or you can manually switch by downloading separate Flutter SDKs.

### Option A: Manual Install (simpler, recommended for beginners)

1. Download Flutter **3.24.5** stable from <https://docs.flutter.dev/get-started/install/windows/desktop>.

    Direct archive link (stable 3.24.5):
    <https://storage.googleapis.com/flutter_infra_release/releases/stable/windows/flutter_windows_3.24.5-stable.zip>

2. Extract to a directory of your choice, for example `C:\flutter\3.24.5\`.

3. Add the `bin` folder to your PATH:

    ```powershell
    # Current session
    $env:Path = "C:\flutter\3.24.5\bin;" + $env:Path
    ```

4. Also download Flutter **3.22.3** stable for the bridge step (Step 13):

    <https://storage.googleapis.com/flutter_infra_release/releases/stable/windows/flutter_windows_3.22.3-stable.zip>

    Extract this to a separate directory, e.g. `C:\flutter\3.22.3\`.

5. Verify the active Flutter version:

    ```powershell
    flutter --version
    # Should show Flutter 3.24.5
    ```

### Option B: Using FVM

```powershell
dart pub global activate fvm
fvm install 3.24.5
fvm install 3.22.3
fvm use 3.24.5
```

> [!NOTE]
> For the remainder of this guide, we assume `flutter` on your PATH points to **3.24.5** initially.
> In Step 13 (Bridge Generation), you will temporarily switch to **3.22.3**.

---

## 9. Replace Flutter Engine with RustDesk Custom Engine

RustDesk uses a custom-built Flutter engine. You must replace the default engine files.

1. First, let Flutter download its default engine artifacts:

    ```powershell
    flutter doctor -v
    flutter precache --windows
    ```

2. Find your Flutter cache directory. It will be at:

    ```
    <your-flutter-sdk-path>\bin\cache\artifacts\engine\windows-x64-release\
    ```

    For example, if Flutter is at `C:\flutter\3.24.5\`:

    ```
    C:\flutter\3.24.5\bin\cache\artifacts\engine\windows-x64-release\
    ```

3. Download and replace the engine:

    ```powershell
    # Download the custom engine
    Invoke-WebRequest -Uri "https://github.com/rustdesk/engine/releases/download/main/windows-x64-release.zip" -OutFile windows-x64-release.zip

    # Extract it
    Expand-Archive -Path windows-x64-release.zip -DestinationPath windows-x64-release

    # Copy all files into the Flutter engine directory (overwriting existing files)
    # IMPORTANT: Replace the path below with YOUR actual Flutter SDK path!
    Copy-Item -Path "windows-x64-release\*" -Destination "C:\flutter\3.24.5\bin\cache\artifacts\engine\windows-x64-release\" -Recurse -Force

    # Cleanup
    Remove-Item -Path windows-x64-release.zip
    Remove-Item -Path windows-x64-release -Recurse
    ```

> [!CAUTION]
> Make sure you replace the path `C:\flutter\3.24.5\` with the actual path to your Flutter **3.24.5** SDK.
> If this step is done incorrectly, the final Flutter build will fail or produce a broken binary.

---

## 10. Patch Flutter

RustDesk applies a patch to Flutter's `dropdown_menu.dart` to fix an `enableFilter` issue.

The patch file is located at: `.github/patches/flutter_3.24.4_dropdown_menu_enableFilter.diff`

1. Copy the patch file to the Flutter SDK root:

    ```powershell
    # Replace with your actual Flutter SDK path
    Copy-Item ".github\patches\flutter_3.24.4_dropdown_menu_enableFilter.diff" -Destination "C:\flutter\3.24.5\"
    ```

2. Navigate to the Flutter SDK root and apply the patch:

    ```powershell
    cd "C:\flutter\3.24.5"
    git apply flutter_3.24.4_dropdown_menu_enableFilter.diff
    ```

3. Navigate back to the RustDesk project directory:

    ```powershell
    cd d:\twekr_mod\rustdesk   # or wherever your project is
    ```

> [!NOTE]
> This patch is specifically for Flutter 3.24.5 (the diff filename says 3.24.4 but the CI
> conditionally applies it when `FLUTTER_VERSION == 3.24.5`). It fixes a late initialization
> issue with the `_enableFilter` variable in `DropdownMenu`.

---

## 11. Install & Setup vcpkg

**vcpkg commit: `120deac3062162151622ca4860575a33844ba10b`**

vcpkg is used to build C/C++ dependencies like `ffmpeg`, `opus`, `libvpx`, `libyuv`, `aom`, etc.

1. Clone vcpkg:

    ```powershell
    git clone https://github.com/microsoft/vcpkg.git C:\vcpkg
    ```

2. Checkout the exact required commit:

    ```powershell
    cd C:\vcpkg
    git checkout 120deac3062162151622ca4860575a33844ba10b
    ```

3. Bootstrap vcpkg:

    ```powershell
    .\bootstrap-vcpkg.bat
    ```

4. Set the required environment variables. **Both variables are crucial:**

    ```powershell
    # For the current session:
    $env:VCPKG_ROOT = "C:\vcpkg"
    $env:VCPKG_DEFAULT_HOST_TRIPLET = "x64-windows-static"

    # To set them permanently:
    [System.Environment]::SetEnvironmentVariable("VCPKG_ROOT", "C:\vcpkg", "User")
    [System.Environment]::SetEnvironmentVariable("VCPKG_DEFAULT_HOST_TRIPLET", "x64-windows-static", "User")
    ```

5. Verify:

    ```powershell
    & "$env:VCPKG_ROOT\vcpkg.exe" version
    ```

> [!WARNING]
> The `VCPKG_DEFAULT_HOST_TRIPLET` environment variable **must** be set to `x64-windows-static`.
> If this is missing, vcpkg will default to `x64-windows` (dynamic linking), and the Rust build
> will fail with linker errors because it expects statically linked libraries.

---

## 12. Install vcpkg Dependencies

Navigate back to the RustDesk repo root, then run:

```powershell
cd d:\twekr_mod\rustdesk

& "$env:VCPKG_ROOT\vcpkg.exe" install --triplet x64-windows-static --x-install-root="$env:VCPKG_ROOT\installed"
```

This reads the `vcpkg.json` manifest file in the project root and installs the required dependencies:

- `aom` (AV1 codec)
- `libjpeg-turbo`
- `opus` (audio codec)
- `libvpx` (VP8/VP9 codec)
- `libyuv` (color conversion)
- `mfx-dispatch` (Intel Media SDK)
- `ffmpeg` (with `amf`, `nvcodec`, `qsv` features for hardware encoding)

> [!NOTE]
> This step takes a **long time** (30–60+ minutes) on the first run, as it compiles ffmpeg and
> other libraries from source. Subsequent runs are fast if you don't change the vcpkg commit.

After completion, verify that the libraries are in place:

```powershell
ls "$env:VCPKG_ROOT\installed\x64-windows-static\lib"
```

You should see `.lib` files for the dependencies listed above.

---

## 13. Generate the Flutter–Rust Bridge

> [!CAUTION]
> This step uses a **different Flutter version** (`3.22.3`) than the rest of the build. It also
> requires a **temporary modification** to `flutter/pubspec.yaml` that **must be reverted** before
> the final build in Step 14. Pay very close attention to the instructions below.

### 13.1. Switch to Flutter 3.22.3

The bridge codegen requires Flutter **3.22.3** (as specified in `bridge.yml`).

```powershell
# If using separate installations, update your PATH:
$env:Path = "C:\flutter\3.22.3\bin;" + $env:Path

# Verify
flutter --version
# Should show Flutter 3.22.3
```

### 13.2. Install Bridge Codegen Dependencies

```powershell
cargo install cargo-expand --version 1.0.95 --locked
cargo install flutter_rust_bridge_codegen --version 1.80.1 --features "uuid" --locked
```

### 13.3. Temporarily Modify `extended_text` Version

The bridge codegen step requires `extended_text` version **13.0.0** in `flutter/pubspec.yaml`. The file currently has version **14.0.0**.

Open `flutter/pubspec.yaml` in a text editor and find this line (around line 108):

```yaml
extended_text: 14.0.0
```

Change it to:

```yaml
extended_text: 13.0.0
```

Then run:

```powershell
cd flutter
flutter pub get
cd ..
```

### 13.4. Ensure Submodules Are Initialized

If you haven't already, make sure all submodules are pulled:

```powershell
git submodule update --init --recursive
```

### 13.5. Run the Bridge Codegen

```powershell
# The codegen binary is in your cargo bin directory
# On Windows this is typically: %USERPROFILE%\.cargo\bin\

flutter_rust_bridge_codegen --rust-input ./src/flutter_ffi.rs --dart-output ./flutter/lib/generated_bridge.dart --c-output ./flutter/macos/Runner/bridge_generated.h
```

This generates/updates the following files:

- `src/bridge_generated.rs`
- `src/bridge_generated.io.rs`
- `flutter/lib/generated_bridge.dart`
- `flutter/lib/generated_bridge.freezed.dart`
- `flutter/macos/Runner/bridge_generated.h`

### 13.6. ⚠️ Revert `extended_text` Version Back to 14.0.0

**This is critical!** The `extended_text` change was only needed for the bridge codegen step. The actual RustDesk build requires version **14.0.0**.

Open `flutter/pubspec.yaml` again and change:

```yaml
extended_text: 13.0.0
```

Back to:

```yaml
extended_text: 14.0.0
```

### 13.7. Switch Back to Flutter 3.24.5

```powershell
# Restore Flutter 3.24.5 on your PATH:
$env:Path = "C:\flutter\3.24.5\bin;" + $env:Path

# Verify
flutter --version
# Should show Flutter 3.24.5
```

> [!IMPORTANT]
> **Why two Flutter versions?** In the CI pipeline, `bridge.yml` and `flutter-build.yml` run as
> **separate, isolated workflows**. The bridge step uses Flutter 3.22.3 and modifies
> `extended_text` to 13.0.0, then uploads the generated bridge files as artifacts. The main
> build workflow downloads those artifacts and builds with Flutter 3.24.5 and `extended_text`
> 14.0.0. Since they are isolated in CI, there's no conflict.
>
> **Locally**, both steps happen in the same working tree, so you must manually handle the
> Flutter version switch and the `extended_text` version toggle. Forgetting to revert will
> cause build failures in Step 14.

---

## 14. Build RustDesk

Make sure you are in the project root directory, Flutter **3.24.5** is on your PATH, and `extended_text` is set back to `14.0.0` in `flutter/pubspec.yaml`.

1. Run `flutter pub get` with the correct Flutter version:

    ```powershell
    cd flutter
    flutter pub get
    cd ..
    ```

2. Run the build:

    ```powershell
    python3 .\build.py --portable --hwcodec --flutter --vram --skip-portable-pack
    ```

    **What the flags do:**
    | Flag | Description |
    |---|---|
    | `--flutter` | Build the Flutter-based UI (instead of the older Sciter UI) |
    | `--hwcodec` | Enable hardware codec support (hardware video encoding/decoding) |
    | `--vram` | Enable VRAM feature (GPU texture rendering, Windows only) |
    | `--portable` | Build a portable (self-contained) Windows executable |
    | `--skip-portable-pack` | Skip creating the portable installer packer (useful for dev) |

3. After a successful build, the output will be at:

    ```
    flutter\build\windows\x64\runner\Release\
    ```

    You can run `rustdesk.exe` directly from that folder.

> [!TIP]
> The first build will take a significant amount of time (15–30+ minutes) because it compiles
> the entire Rust codebase. Subsequent builds (after code changes) will be much faster due to
> incremental compilation.

---

## 15. Quick Reference — All Versions at a Glance

| Component                         | Version / Value                                                                 |
| --------------------------------- | ------------------------------------------------------------------------------- |
| Base RustDesk Commit              | `998b75856da4199ac009ce4135b4fba1b48099e6`                                      |
| LLVM                              | `15.0.6`                                                                        |
| Rust Toolchain                    | `1.75`                                                                          |
| Rust Target                       | `x86_64-pc-windows-msvc`                                                        |
| Flutter (Build)                   | `3.24.5` (stable)                                                               |
| Flutter (Bridge Codegen)          | `3.22.3` (stable)                                                               |
| vcpkg Commit                      | `120deac3062162151622ca4860575a33844ba10b`                                      |
| vcpkg Triplet                     | `x64-windows-static`                                                            |
| `cargo-expand`                    | `1.0.95`                                                                        |
| `flutter_rust_bridge_codegen`     | `1.80.1`                                                                        |
| `extended_text` (for bridge step) | `13.0.0`                                                                        |
| `extended_text` (for build step)  | `14.0.0`                                                                        |
| Flutter Engine                    | Custom from `rustdesk/engine` (branch: `main`)                                  |
| Flutter Patch                     | `flutter_3.24.4_dropdown_menu_enableFilter.diff`                                |
| Build Command                     | `python3 .\build.py --portable --hwcodec --flutter --vram --skip-portable-pack` |

---

## Troubleshooting

### `LIBCLANG_PATH` not found

Make sure LLVM is installed and `LIBCLANG_PATH` is set to the LLVM `bin` directory (e.g. `C:\Program Files\LLVM\bin`).

### Linker errors about missing `.lib` files

Ensure `VCPKG_ROOT` is set and `VCPKG_DEFAULT_HOST_TRIPLET` is set to `x64-windows-static`. Re-run the vcpkg install command from Step 12.

### Bridge codegen fails with `hbb_common` errors

Run `git submodule update --init --recursive` to initialize the `hbb_common` submodule.

### `extended_text` version conflict

If you see errors about `extended_text` during `flutter pub get` or the build:

- For the **bridge codegen step** (Step 13): it should be `13.0.0`
- For the **build step** (Step 14): it should be `14.0.0`

### Flutter engine crash or `flutter build windows` errors

Make sure you followed Step 9 correctly — the custom engine files must replace the originals at `<flutter-sdk>/bin/cache/artifacts/engine/windows-x64-release/`.

### `python3` not found on Windows

Try using `python` instead, or check that Python is installed and on your PATH. The Windows Store alias for `python3` might need to be enabled in system settings under **"Manage App Execution Aliases"**.
