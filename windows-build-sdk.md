# Windows Build SDK

This document describes two blocks for creating a Windows Build SDK:

1. build infrastructure using WSL (Windows Subsystem for Linux) with a Yocto-built rootfs and MSVC
   toolchain for creating a reproducible build environment with all compilers, libraries, and tools
   for building Erlang/OTP binaries from source code.
2. runtime infrastructure which describes where and how the build environment will be executed.
3. code signing via Azure Trusted Signing for producing distributable Windows binaries.

## Build Infrastructure

The following describes what is called _build infrastructure_ which is a set of configurations used
for creating all tools needed to build Erlang/OTP binaries for Windows platforms.

We use **WSL as the Unix-y driver with MSVC as the C compiler**. This is the official OTP Windows
build approach as documented in OTP's INSTALL-WIN32 instructions. The build process uses bash and
make running in WSL, while the emulator C code is compiled with Microsoft Visual C++, yielding
native Windows binaries.

For the WSL environment, we use a **Yocto-built rootfs** from the `meta-erlang-build` layer - the
same infrastructure used for Linux builds. This provides consistency across platforms and full
control over the build environment.

This approach:

 - Matches upstream OTP's official Windows build process
 - Uses WSL with Yocto rootfs to provide reproducible Unix build tools (bash, make, autoconf)
 - Uses MSVC/Windows SDK for native Windows compilation
 - Produces official-format binaries (`otp_win64_XX.exe`)
 - Shares build infrastructure with Linux builds via `meta-erlang-build`

As a framework this setup gives us some important non-functional requirements:

 - support for x86-64 Windows architecture
 - the capability to pin exact versions of all dependencies via Yocto
 - SBOM generation for the build environment itself (from Yocto)
 - builds fully isolated from host system drift
 - consistency with Linux build toolchain

And the following functional requirements:

 - produce an Erlang/OTP NSIS installer (`otp_win64_XX.exe`) matching OTP naming conventions
 - produce binaries that run on Windows 10 and later
 - have configuration for x86-64 architecture
 - possibility to tune build flags for specific requirements

### Version and Dependency Selection

For reproducibility and stability, we use a Yocto-built rootfs from the `meta-erlang-build` layer
to provide the build tools. The Yocto release `scarthgap` (5.0) provides pinned versions of:

 - GNU make
 - autoconf, automake
 - Perl
 - mingw-w64 cross-compiler

Note: The libraries in the Yocto rootfs (OpenSSL, ncurses, etc.) are not used - the final Windows
binaries link against Windows-native libraries. The Yocto rootfs only provides reproducible build
tools.

For the Windows-side components:

 - Pin the GitHub Actions runner to `windows-2022` (not `windows-latest`) to lock Visual Studio
   and Windows SDK versions
 - NSIS installed via Chocolatey with pinned version

The `windows-2022` runner provides:

 - Visual Studio 2022 with MSVC toolchain
 - Windows SDK
 - WSL 2 support

### WSL Configuration with Yocto Rootfs

The build uses a Yocto-built rootfs tarball imported as a custom WSL distribution. This rootfs
is the same x86-64 image used for Linux builds, providing:

 - All build tools pre-installed (gcc, make, autoconf, automake, perl)
 - mingw-w64 cross-compiler for Windows targets
 - Pinned versions of all dependencies
 - No runtime package installation needed

The rootfs is imported using `wsl --import`:

```powershell
# Download rootfs from OCI registry
oras pull ghcr.io/erlef/erlang-builds/build-env:x86-64-glibc -o rootfs.tar

# Import as WSL distribution
wsl --import erlang-build C:\wsl\erlang-build rootfs.tar
```

The OTP source is placed on the Windows filesystem (`/mnt/c/...`) so that Windows tooling
(MSVC, NSIS) can access the files directly.

### Library Dependencies

The final Windows binaries link against Windows-native libraries, not libraries from the WSL
environment. These can be installed via Chocolatey with pinned versions for reproducibility:

```powershell
choco install openssl --version=3.4.1 -y
choco install wxwidgets --version=3.2.6 -y
```

Required dependencies:

 - [OpenSSL for Windows](https://community.chocolatey.org/packages/openssl) (for `crypto`, `ssl` applications)
 - [wxWidgets for Windows](https://community.chocolatey.org/packages/wxwidgets) (for `wx` application / Observer)
 - ODBC (Windows native, included in Windows SDK)

### NSIS Installer

NSIS (Nullsoft Scriptable Install System) is required to produce the official Windows installer.
NSIS must be installed on the Windows side (not in WSL) and accessible from the build process.

Install NSIS via Chocolatey with a pinned version for reproducibility:

```powershell
choco install nsis --version=3.09 -y
```

### SBOM Generation

For SBOM (Software Bill of Materials) generation:

 - The Yocto build provides SBOM and license reports for all components in the rootfs
 - Windows-compatible SBOM tooling generates CycloneDX/SPDX for the final OTP build
 - Integration with the project's SBOM management infrastructure (Dependency Track)

## Runtime Infrastructure

Given that the previous section takes care about producing a reproducible build environment,
this section describes how we execute the build.

### GitHub Actions Integration

For CI/CD, we use GitHub-hosted Windows runners with WSL:

 - `windows-2022` runner (pinned for stability)
 - Yocto rootfs imported as custom WSL distribution
 - Build commands executed in WSL, with output on Windows filesystem

### Example GitHub Actions Workflow

```yaml
name: Build Erlang/OTP for Windows

on:
  workflow_dispatch:
    inputs:
      otp_version:
        description: 'OTP version to build'
        required: true
        type: string

jobs:
  build:
    runs-on: windows-2022

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      # Download Yocto-built rootfs from OCI registry
      - name: Download build environment
        run: |
          # Install ORAS CLI
          Invoke-WebRequest -Uri "https://github.com/oras-project/oras/releases/download/v1.2.0/oras_1.2.0_windows_amd64.zip" -OutFile oras.zip
          Expand-Archive oras.zip -DestinationPath oras

          # Pull rootfs from registry
          .\oras\oras.exe pull ghcr.io/erlef/erlang-builds/build-env:x86-64-glibc -o rootfs.tar

      # Import as WSL distribution
      - name: Setup WSL with Yocto rootfs
        run: |
          wsl --import erlang-build C:\wsl\erlang-build rootfs.tar
          wsl -d erlang-build -- uname -a

      # Install NSIS on Windows side (pinned version)
      - name: Install NSIS
        run: choco install nsis --version=3.09 -y

      # Build OTP in WSL
      - name: Build Erlang/OTP
        run: |
          wsl -d erlang-build -- bash -c '
            set -euo pipefail

            # Create build directory on Windows filesystem
            mkdir -p /mnt/c/src
            cd /mnt/c/src

            # Download OTP source
            curl -LO https://github.com/erlang/otp/releases/download/OTP-${{ inputs.otp_version }}/otp_src_${{ inputs.otp_version }}.tar.gz
            tar xzf otp_src_${{ inputs.otp_version }}.tar.gz
            cd otp_src_${{ inputs.otp_version }}

            # Build OTP following INSTALL-WIN32 instructions
            eval $(./otp_build env_win32 x64)
            ./otp_build autoconf
            ./otp_build configure
            ./otp_build boot -a
            ./otp_build release -a /mnt/c/src/release
            ./otp_build installer_win32
          '

      - name: Upload installer artifact
        uses: actions/upload-artifact@v4
        with:
          name: erlang-${{ inputs.otp_version }}-win64-installer
          path: C:\src\release\otp_win64_*.exe

      - name: Upload release artifact
        uses: actions/upload-artifact@v4
        with:
          name: erlang-${{ inputs.otp_version }}-win64
          path: C:\src\release\
```

### Build Process Overview

The OTP Windows build follows these steps (as per INSTALL-WIN32):

1. **Environment setup**: `eval $(./otp_build env_win32 x64)` sets up paths for MSVC
2. **Autoconf**: `./otp_build autoconf` generates configure scripts
3. **Configure**: `./otp_build configure` configures the build
4. **Boot**: `./otp_build boot -a` builds the system
5. **Release**: `./otp_build release -a <dest>` creates the release
6. **Installer**: `./otp_build installer_win32` creates the NSIS installer

## Code Signing

For distributable Windows binaries, code signing is required. Per project requirements, we use
Azure Trusted Signing with GitHub Actions OIDC token authentication.

Authentication uses the `azure/login` action with OIDC token exchange, avoiding long-lived
credentials. The signing itself uses `azure/trusted-signing-action`.

For reference implementation, see the
[Elixir release workflow](https://github.com/elixir-lang/elixir/blob/main/.github/workflows/release.yml).

## Reproducibility Considerations

### What is Reproducible

With this setup, we achieve _functional reproducibility_:

 - Yocto-built rootfs = fully controlled build environment with pinned toolchain
 - Same rootfs as Linux builds = consistent toolchain across platforms
 - Pinned `windows-2022` runner = same Visual Studio and Windows SDK versions
 - Pinned NSIS version via Chocolatey = consistent installer generation
 - Same OTP source = same build output

### Strategies for Stability

1. **Yocto rootfs**: The `meta-erlang-build` layer provides a fully reproducible build environment
   with all tools pinned. No runtime package installation means no drift.

2. **Pin the runner image**: Using `windows-2022` instead of `windows-latest` prevents
   MSVC/SDK drift.

3. **Pin Windows tool versions**: Use explicit versions with Chocolatey (e.g., `nsis --version=3.09`).

4. **Cache the rootfs**: Store the Yocto rootfs in OCI registry for fast, consistent access.

### Shared Infrastructure with Linux

The Windows build shares the `meta-erlang-build` Yocto layer with Linux builds for the build
tools (make, autoconf, perl, etc.). This ensures consistent build orchestration across platforms,
though the final linked libraries differ (Windows-native vs Linux).

### Limitations

Full bit-for-bit reproducibility on Windows is challenging due to:

 - Timestamps embedded by MSVC compiler
 - Code signing includes timestamps
 - NSIS installer may embed build-time information
 - Windows PE format includes various metadata

For our purposes, functional reproducibility (same inputs produce equivalent, working binaries)
is acceptable and practical.

### Verification

To verify build consistency:

```powershell
# Compare dependency versions between builds
dumpbin /dependents build1\bin\erl.exe
dumpbin /dependents build2\bin\erl.exe

# Verify the installer was created
Get-ChildItem C:\src\release\otp_win64_*.exe
```

## Artifact Outputs

### Primary Output: NSIS Installer

The main artifact is the NSIS installer following OTP naming conventions:

 - `otp_win64_<version>.exe` - Official Windows installer

This is the standard format used by OTP and expected by downstream tools like `setup-beam`.

### Secondary Output: Portable Archive

Optionally, a ZIP archive of the release directory can be produced for consumers who need
unpacked binaries:

 - `erlang-<version>-win64.zip` - Portable release archive

## Future Considerations

 - **ARM64 Windows**: Windows on ARM support could be added when GitHub provides ARM64 Windows
   runners (blocked on [actions/partner-runner-images#109](https://github.com/actions/partner-runner-images/issues/109))
   and OTP supports ARM64 Windows builds
 - **Additional build flavors**: Different OpenSSL versions, static linking options as needed
 - **MSYS2 alternative**: While WSL+MSVC is the official approach, MSYS2 could be explored as
   an alternative if WSL proves problematic in CI
