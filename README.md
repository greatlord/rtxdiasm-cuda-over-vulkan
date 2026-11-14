# rtxdiasm-cuda-over-vulkan

Workspace repository for building ACuda CUDA-over-Vulkan components and ACuda tests.

## Layout
- `rtxdiasm-modules/` (git submodules)
  - `rtxdiasm-ptx-to-spirv` — PTX → SPIR-V compiler
  - `rtxdiasm_acuda` — CUDA driver API + runtime foundation (`libcuda.so.1` / `nvcuda.dll`)
  - `rtxdiasm-cudart` — CUDA runtime API shim (`cudart*`)
  - `rtxdiasm-nvapi` — NVAPI library (Windows-only: `nvapi64.dll` / `nvapi.dll`)
- `rtxdiasm-tests/rtxdiasm_tests` (git submodule) — unified tests tree (includes ACuda tests used by top-level CMake)

## Build
```bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build --config Release
ctest --test-dir build -C Release --output-on-failure
```

Notes:
- Requires Vulkan and a C toolchain (MSVC on Windows, GCC/Clang on Linux).
- `rtxdiasm-nvapi` is added/built on Windows only.
- ACuda tests are wired from `rtxdiasm-tests/rtxdiasm_tests/tests/acuda` (CTest targets; availability depends on platform/support).
