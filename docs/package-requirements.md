# Package requirements (Linux)

System packages commonly required to build **rtxdiasm-cuda-over-vulkan** on Linux. Package names can vary by release; adjust as needed for your distribution version.

> **Notes**
> - Initialize submodules before building:
>   ```bash
>   git submodule update --init --recursive
>   ```
> - **.NET 8 SDK** is required because the top-level CMake build integrates/builds `rtxdiasm-runtime` (C# SASS→PTX lift runtime/tooling and `isa_json` assets). Install via your distro or Microsoft’s official instructions:
>   - Debian/Ubuntu: https://learn.microsoft.com/dotnet/core/install/linux-debian / https://learn.microsoft.com/dotnet/core/install/linux-ubuntu
>   - Fedora: https://learn.microsoft.com/dotnet/core/install/linux-fedora
> - `lz4` and `zstd` are vendored in submodules (`rtxdiasm-modules/*/external/lz4`, `.../external/zstd`), so system `liblz4`/`libzstd` development packages are not typically required.
> - A Vulkan-capable driver/ICD is needed to run ACuda/Vulkan workloads and tests (build-time headers/loader are separate from runtime drivers).

## Debian

```bash
sudo apt update
sudo apt install -y \
  git \
  build-essential \
  cmake \
  pkg-config \
  libvulkan-dev \
  vulkan-tools \
  glslc \
  ca-certificates \
  curl
```

**Notes**
- If `glslc` is not available as `glslc`, install it from a shaderc/Vulkan SDK package that provides the `glslc` binary and ensure it is on `PATH`.
- Install `dotnet-sdk-8.0` from Microsoft packages if not available in your configured repos.

## Ubuntu

```bash
sudo apt update
sudo apt install -y \
  git \
  build-essential \
  cmake \
  pkg-config \
  libvulkan-dev \
  vulkan-tools \
  glslc \
  ca-certificates \
  curl
```

**Notes**
- For 32-bit/multilib builds you may also need packages such as `gcc-multilib`, `g++-multilib`, and 32-bit Vulkan loader/dev packages (e.g. `libvulkan-dev:i386`, depending on enabled architectures).
- Install `dotnet-sdk-8.0` via Microsoft’s documented package feed for your Ubuntu release if needed.

## Fedora

```bash
sudo dnf install -y \
  git \
  gcc \
  gcc-c++ \
  make \
  cmake \
  pkgconf-pkg-config \
  vulkan-loader-devel \
  vulkan-tools \
  glslc \
  ca-certificates \
  curl
```

**Notes**
- If `glslc` is provided by a different package on your Fedora version, install the package that provides the `glslc` binary.
- For 32-bit/multilib targets you may need `glibc-devel.i686`, 32-bit toolchain support, and `vulkan-loader-devel.i686` (as applicable).
- Install `dotnet-sdk-8.0` as needed for your Fedora release.

## Verify tools

```bash
git --version
cmake --version
gcc --version
g++ --version || true
dotnet --version        # 8.x SDK expected
glslc --version         # used for shader compilation
pkg-config --modversion vulkan || true
vulkaninfo --summary    # optional: verifies Vulkan runtime/driver visibility
```

## Build (reference)

```bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build --config Release -j
ctest --test-dir build -C Release --output-on-failure
```

Outputs are placed under `bin/<config>/<platform-arch>/` (see repository `README.md`). WSL2/device ICD limitations may affect runtime/test execution; ensure a Vulkan 1.3-capable environment where required by ACuda components.
