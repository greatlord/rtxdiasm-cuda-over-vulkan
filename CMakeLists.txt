cmake_minimum_required(VERSION 3.21)
project(rtxdiasm_cuda_over_vulkan LANGUAGES C)

set(CMAKE_C_STANDARD 17)
set(CMAKE_C_STANDARD_REQUIRED ON)

# Build type is recommended for single-config generators.
if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  set(CMAKE_BUILD_TYPE Release CACHE STRING "Build type" FORCE)
endif()

# Output layout: bin/<config>/<platform-arch>
# Linux: bin/<config>/<platform-arch>/{lib,bin,test} (no .a archives in output)
# Windows: bin/<config>/<platform-arch>/{bin,test} (DLLs/exes in bin, tests in test; no .lib/.exp in release/debug)
if(WIN32)
  set(_rtxcov_os win)
  if(CMAKE_GENERATOR_PLATFORM STREQUAL "Win32")
    set(_rtxcov_arch x86)
  elseif(CMAKE_GENERATOR_PLATFORM STREQUAL "ARM64")
    set(_rtxcov_arch arm64)
  elseif(CMAKE_GENERATOR_PLATFORM STREQUAL "ARM")
    set(_rtxcov_arch arm)
  elseif(CMAKE_SIZEOF_VOID_P EQUAL 4)
    set(_rtxcov_arch x86)
  else()
    set(_rtxcov_arch x64)
  endif()
elseif(APPLE)
  set(_rtxcov_os macos)
  if(CMAKE_SYSTEM_PROCESSOR MATCHES "arm64|aarch64|ARM64")
    set(_rtxcov_arch arm64)
  elseif(CMAKE_SIZEOF_VOID_P EQUAL 4)
    set(_rtxcov_arch x86)
  else()
    set(_rtxcov_arch x64)
  endif()
else()
  set(_rtxcov_os linux)
  if(CMAKE_SYSTEM_PROCESSOR MATCHES "aarch64|arm64|ARM64")
    set(_rtxcov_arch arm64)
  elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "arm" OR CMAKE_SIZEOF_VOID_P EQUAL 4)
    set(_rtxcov_arch x86)
  else()
    set(_rtxcov_arch x64)
  endif()
endif()
set(RTXCOV_PLATFORM "${_rtxcov_os}-${_rtxcov_arch}")

set(RTXCOV_OUTPUT_ROOT_RELEASE ${PROJECT_SOURCE_DIR}/bin/release/${RTXCOV_PLATFORM})
set(RTXCOV_OUTPUT_ROOT_DEBUG   ${PROJECT_SOURCE_DIR}/bin/debug/${RTXCOV_PLATFORM})

# Single-config generators honor these; multi-config generators use per-config dirs below.
if(CMAKE_BUILD_TYPE STREQUAL "Debug")
  set(_rtxcov_out_root ${RTXCOV_OUTPUT_ROOT_DEBUG})
  set(_rtxcov_out_cfg "debug")
else()
  set(_rtxcov_out_root ${RTXCOV_OUTPUT_ROOT_RELEASE})
  set(_rtxcov_out_cfg "release")
endif()

if(WIN32)
  # Windows: separate bin/test; keep static/import libraries (.lib/.exp) out of release/debug outputs.
  set(RTXCOV_OUTPUT_DIR ${_rtxcov_out_root})
  set(RTXCOV_RUNTIME_DIR ${RTXCOV_OUTPUT_DIR}/bin)
  set(RTXCOV_LIBRARY_DIR ${RTXCOV_OUTPUT_DIR}/lib)
  set(RTXCOV_TEST_DIR ${RTXCOV_OUTPUT_DIR}/test)
  set(RTXCOV_ARCHIVE_DIR ${PROJECT_BINARY_DIR}/archive/${_rtxcov_out_cfg}/${RTXCOV_PLATFORM})
else()
  # Linux/macOS: separate lib/bin/test; keep static archives (.a) out of release/debug outputs.
  set(RTXCOV_OUTPUT_DIR ${_rtxcov_out_root})
  set(RTXCOV_RUNTIME_DIR ${RTXCOV_OUTPUT_DIR}/bin)
  set(RTXCOV_LIBRARY_DIR ${RTXCOV_OUTPUT_DIR}/lib)
  set(RTXCOV_TEST_DIR ${RTXCOV_OUTPUT_DIR}/test)
  set(RTXCOV_ARCHIVE_DIR ${PROJECT_BINARY_DIR}/archive/${_rtxcov_out_cfg}/${RTXCOV_PLATFORM})
endif()

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${RTXCOV_RUNTIME_DIR})
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${RTXCOV_LIBRARY_DIR})
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${RTXCOV_ARCHIVE_DIR})
foreach(_cfg DEBUG RELEASE RELWITHDEBINFO MINSIZEREL)
  if(_cfg STREQUAL "DEBUG")
    set(_cfg_root ${RTXCOV_OUTPUT_ROOT_DEBUG})
    set(_cfg_name "debug")
  else()
    set(_cfg_root ${RTXCOV_OUTPUT_ROOT_RELEASE})
    set(_cfg_name "release")
  endif()
  set(_cfg_runtime ${_cfg_root}/bin)
  set(_cfg_library ${_cfg_root}/lib)
  set(_cfg_archive ${PROJECT_BINARY_DIR}/archive/${_cfg_name}/${RTXCOV_PLATFORM})
  set(CMAKE_RUNTIME_OUTPUT_DIRECTORY_${_cfg} ${_cfg_runtime})
  set(CMAKE_LIBRARY_OUTPUT_DIRECTORY_${_cfg} ${_cfg_library})
  set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY_${_cfg} ${_cfg_archive})
endforeach()
unset(_cfg)
unset(_cfg_root)
unset(_cfg_name)
unset(_cfg_runtime)
unset(_cfg_library)
unset(_cfg_archive)
unset(_rtxcov_out_root)
unset(_rtxcov_out_cfg)
unset(_rtxcov_os)
unset(_rtxcov_arch)

if(MSVC)
  set(ACUDA_WARNINGS /W3)
  add_compile_definitions(_CRT_SECURE_NO_WARNINGS)
else()
  set(ACUDA_WARNINGS -Wall -Wextra -Werror)
endif()

find_package(Vulkan REQUIRED)

# Paths
set(RTX_MODULES_DIR ${PROJECT_SOURCE_DIR}/rtxdiasm-modules)
set(RTX_TESTS_ROOT  ${PROJECT_SOURCE_DIR}/rtxdiasm-tests/rtxdiasm_tests)
set(ACUDA_TESTS_DIR ${RTX_TESTS_ROOT}/tests/acuda)

# Compiler module path (used by acuda module/tests)
if(NOT DEFINED RTXDIASM_PTX_TO_SPIRV_DIR)
  set(RTXDIASM_PTX_TO_SPIRV_DIR ${RTX_MODULES_DIR}/rtxdiasm-ptx-to-spirv)
endif()

# --- Modules ---
# Shared fatbin PTX parse/extract (lz4+zstd)
add_subdirectory(${RTX_MODULES_DIR}/rtxdiasm-fatbin-parse ${CMAKE_BINARY_DIR}/rtxdiasm-fatbin-parse EXCLUDE_FROM_ALL)

# ACuda driver + runtime (libcuda.so.1 / nvcuda.dll, acuda_runtime)
# (brings in PTX->SPIR-V compiler dependency from sibling module as needed)
add_subdirectory(${RTX_MODULES_DIR}/rtxdiasm_acuda ${CMAKE_BINARY_DIR}/rtxdiasm_acuda EXCLUDE_FROM_ALL)

# CUDA Runtime API (cudart)
if(DEFINED ACUDA_DRIVER_TRACE_SOURCE AND NOT "${ACUDA_DRIVER_TRACE_SOURCE}" STREQUAL "")
  set(ACUDA_TRACE_SOURCE "${ACUDA_DRIVER_TRACE_SOURCE}")
else()
  set(ACUDA_TRACE_SOURCE "")
endif()
add_subdirectory(${RTX_MODULES_DIR}/rtxdiasm-cudart ${CMAKE_BINARY_DIR}/rtxdiasm-cudart EXCLUDE_FROM_ALL)
if(NOT TARGET acuda_cudart AND TARGET rtxcudart)
  add_library(acuda_cudart ALIAS rtxcudart)
endif()

# NVAPI (Windows-only)
if(WIN32)
  # Point nvapi at acuda driver headers/target in this layout.
  set(ACUDA_ROOT ${RTX_MODULES_DIR}/rtxdiasm_acuda CACHE PATH "Path to acuda module root" FORCE)
  set(ACUDA_INCLUDE_DIR ${RTX_MODULES_DIR}/rtxdiasm_acuda/include CACHE PATH "Path to acuda public headers" FORCE)
  add_subdirectory(${RTX_MODULES_DIR}/rtxdiasm-nvapi ${CMAKE_BINARY_DIR}/rtxdiasm-nvapi EXCLUDE_FROM_ALL)
  # NVAPI submodule sets its own output dirs; align with workspace layout.
  foreach(_nvapi_tgt IN ITEMS nvapi nvapi64_loader acuda_nvapi)
    if(TARGET ${_nvapi_tgt})
      set_target_properties(${_nvapi_tgt} PROPERTIES
        RUNTIME_OUTPUT_DIRECTORY ${RTXCOV_RUNTIME_DIR}
        LIBRARY_OUTPUT_DIRECTORY ${RTXCOV_LIBRARY_DIR}
        ARCHIVE_OUTPUT_DIRECTORY ${RTXCOV_ARCHIVE_DIR})
      foreach(_cfg DEBUG RELEASE RELWITHDEBINFO MINSIZEREL)
        if(_cfg STREQUAL "DEBUG")
          set(_cfg_root ${RTXCOV_OUTPUT_ROOT_DEBUG})
          set(_cfg_name "debug")
        else()
          set(_cfg_root ${RTXCOV_OUTPUT_ROOT_RELEASE})
          set(_cfg_name "release")
        endif()
        set_target_properties(${_nvapi_tgt} PROPERTIES
          RUNTIME_OUTPUT_DIRECTORY_${_cfg} ${_cfg_root}/bin
          LIBRARY_OUTPUT_DIRECTORY_${_cfg} ${_cfg_root}/lib
          ARCHIVE_OUTPUT_DIRECTORY_${_cfg} ${PROJECT_BINARY_DIR}/archive/${_cfg_name}/${RTXCOV_PLATFORM})
      endforeach()
      unset(_cfg)
      unset(_cfg_root)
      unset(_cfg_name)
    endif()
  endforeach()
  unset(_nvapi_tgt)
endif()

# Output directories are set globally above (bin/<config>/<platform-arch>).

# --- rtxdiasm-runtime (C# lift backend: rtxdisam_native/host/cli + isa_json) ---
# Built/published via dotnet + CMake (host) so ACuda's rtxdisam bridge can find it
# alongside other runtime binaries in bin/<config>/<platform>/bin.
set(RTXDIASM_RUNTIME_DIR ${RTX_MODULES_DIR}/rtxdiasm-runtime)
if(EXISTS "${RTXDIASM_RUNTIME_DIR}/src/rtxdisam.native/rtxdisam.native.csproj")
  find_program(DOTNET_EXECUTABLE dotnet)
  if(DOTNET_EXECUTABLE)
    # Map platform/arch to .NET RID
    if(WIN32)
      if(CMAKE_SIZEOF_VOID_P EQUAL 4)
        set(RTXDIASM_RUNTIME_RID win-x86)
      else()
        set(RTXDIASM_RUNTIME_RID win-x64)
      endif()
    elseif(APPLE)
      if(CMAKE_SYSTEM_PROCESSOR MATCHES "arm64|aarch64|ARM64")
        set(RTXDIASM_RUNTIME_RID osx-arm64)
      else()
        set(RTXDIASM_RUNTIME_RID osx-x64)
      endif()
    else()
      if(CMAKE_SYSTEM_PROCESSOR MATCHES "aarch64|arm64|ARM64")
        set(RTXDIASM_RUNTIME_RID linux-arm64)
      elseif(CMAKE_SIZEOF_VOID_P EQUAL 4)
        set(RTXDIASM_RUNTIME_RID linux-x86)
      else()
        set(RTXDIASM_RUNTIME_RID linux-x64)
      endif()
    endif()

    set(RTXDIASM_RUNTIME_ARTIFACTS ${RTXDIASM_RUNTIME_DIR}/artifacts/${RTXDIASM_RUNTIME_RID})
    set(RTXDIASM_RUNTIME_HOST_BUILD ${CMAKE_BINARY_DIR}/rtxdiasm-runtime-host)

    set(_rtxrt_stamp ${CMAKE_BINARY_DIR}/rtxdiasm-runtime.stamp)
    # Build/publish Release artifacts once and copy into release+debug output bin dirs.
    set(_rtxrt_cmds
      COMMAND ${CMAKE_COMMAND} -E echo "Building rtxdiasm-runtime (${RTXDIASM_RUNTIME_RID})"
      COMMAND ${DOTNET_EXECUTABLE} publish "${RTXDIASM_RUNTIME_DIR}/src/rtxdisam.native/rtxdisam.native.csproj" -c Release -r ${RTXDIASM_RUNTIME_RID} -o "${RTXDIASM_RUNTIME_ARTIFACTS}/native"
      COMMAND ${DOTNET_EXECUTABLE} publish "${RTXDIASM_RUNTIME_DIR}/src/rtxdisam.cli/rtxdisam.cli.csproj" -c Release -r ${RTXDIASM_RUNTIME_RID} -o "${RTXDIASM_RUNTIME_ARTIFACTS}/cli"
      COMMAND ${CMAKE_COMMAND} -E make_directory "${RTXCOV_OUTPUT_ROOT_RELEASE}/bin" "${RTXCOV_OUTPUT_ROOT_DEBUG}/bin"
      COMMAND ${CMAKE_COMMAND} -E copy_directory "${RTXDIASM_RUNTIME_ARTIFACTS}/native" "${RTXCOV_OUTPUT_ROOT_RELEASE}/bin"
      COMMAND ${CMAKE_COMMAND} -E copy_directory "${RTXDIASM_RUNTIME_ARTIFACTS}/native" "${RTXCOV_OUTPUT_ROOT_DEBUG}/bin"
      COMMAND ${CMAKE_COMMAND} -E copy_directory "${RTXDIASM_RUNTIME_ARTIFACTS}/cli" "${RTXCOV_OUTPUT_ROOT_RELEASE}/bin"
      COMMAND ${CMAKE_COMMAND} -E copy_directory "${RTXDIASM_RUNTIME_ARTIFACTS}/cli" "${RTXCOV_OUTPUT_ROOT_DEBUG}/bin"
      COMMAND ${CMAKE_COMMAND} -E copy_directory "${RTXDIASM_RUNTIME_DIR}/isa_json" "${RTXCOV_OUTPUT_ROOT_RELEASE}/bin/isa_json"
      COMMAND ${CMAKE_COMMAND} -E copy_directory "${RTXDIASM_RUNTIME_DIR}/isa_json" "${RTXCOV_OUTPUT_ROOT_DEBUG}/bin/isa_json"
    )
    if(WIN32)
      # rtxdisam_host is currently Windows-specific (Win32 API). Build/copy on Windows only.
      set(RTXDIASM_RUNTIME_HOST_BUILD ${CMAKE_BINARY_DIR}/rtxdiasm-runtime-host)
      if(CMAKE_CONFIGURATION_TYPES OR MSVC)
        set(_rtxrt_host_exe "${RTXDIASM_RUNTIME_HOST_BUILD}/Release/rtxdisam_host${CMAKE_EXECUTABLE_SUFFIX}")
      else()
        set(_rtxrt_host_exe "${RTXDIASM_RUNTIME_HOST_BUILD}/rtxdisam_host${CMAKE_EXECUTABLE_SUFFIX}")
      endif()
      list(APPEND _rtxrt_cmds
        COMMAND ${CMAKE_COMMAND} -S "${RTXDIASM_RUNTIME_DIR}/src/rtxdisam.host" -B "${RTXDIASM_RUNTIME_HOST_BUILD}" -DCMAKE_BUILD_TYPE=Release
        COMMAND ${CMAKE_COMMAND} --build "${RTXDIASM_RUNTIME_HOST_BUILD}" --config Release
        COMMAND ${CMAKE_COMMAND} -E copy_if_different "${_rtxrt_host_exe}" "${RTXCOV_OUTPUT_ROOT_RELEASE}/bin/rtxdisam_host${CMAKE_EXECUTABLE_SUFFIX}"
        COMMAND ${CMAKE_COMMAND} -E copy_if_different "${_rtxrt_host_exe}" "${RTXCOV_OUTPUT_ROOT_DEBUG}/bin/rtxdisam_host${CMAKE_EXECUTABLE_SUFFIX}"
      )
      unset(_rtxrt_host_exe)
    endif()
    list(APPEND _rtxrt_cmds
      COMMAND ${CMAKE_COMMAND} -E rm -f
        "${RTXCOV_OUTPUT_ROOT_RELEASE}/bin/rtxdisam.core.pdb"
        "${RTXCOV_OUTPUT_ROOT_RELEASE}/bin/rtxdisam.pdb"
        "${RTXCOV_OUTPUT_ROOT_RELEASE}/bin/rtxdisam_native.pdb"
        "${RTXCOV_OUTPUT_ROOT_RELEASE}/bin/rtxdisam_native.exp"
        "${RTXCOV_OUTPUT_ROOT_DEBUG}/bin/rtxdisam.core.pdb"
        "${RTXCOV_OUTPUT_ROOT_DEBUG}/bin/rtxdisam.pdb"
        "${RTXCOV_OUTPUT_ROOT_DEBUG}/bin/rtxdisam_native.pdb"
        "${RTXCOV_OUTPUT_ROOT_DEBUG}/bin/rtxdisam_native.exp"
      COMMAND ${CMAKE_COMMAND} -E touch ${_rtxrt_stamp}
    )
    add_custom_command(
      OUTPUT ${_rtxrt_stamp}
      ${_rtxrt_cmds}
      WORKING_DIRECTORY "${RTXDIASM_RUNTIME_DIR}"
      VERBATIM
    )
    unset(_rtxrt_cmds)
    add_custom_target(rtxdiasm_runtime ALL DEPENDS ${_rtxrt_stamp})
    unset(_rtxrt_stamp)
  else()
    message(STATUS "dotnet not found; skipping rtxdiasm-runtime build (rtxdisam lift backend)")
  endif()
endif()

# --- ACuda tests (from rtxdiasm_tests module) ---
include(CTest)

function(rtxcov_add_test name source)
  add_executable(${name} ${source})
  target_link_libraries(${name} PRIVATE acuda_runtime)
  target_compile_options(${name} PRIVATE ${ACUDA_WARNINGS})
  if(DEFINED RTXCOV_TEST_DIR)
    set_target_properties(${name} PROPERTIES RUNTIME_OUTPUT_DIRECTORY ${RTXCOV_TEST_DIR})
    foreach(_cfg DEBUG RELEASE RELWITHDEBINFO MINSIZEREL)
      if(_cfg STREQUAL "DEBUG")
        set_target_properties(${name} PROPERTIES RUNTIME_OUTPUT_DIRECTORY_${_cfg} ${RTXCOV_OUTPUT_ROOT_DEBUG}/test)
      else()
        set_target_properties(${name} PROPERTIES RUNTIME_OUTPUT_DIRECTORY_${_cfg} ${RTXCOV_OUTPUT_ROOT_RELEASE}/test)
      endif()
    endforeach()
    unset(_cfg)
  endif()
  add_test(NAME ${name} COMMAND ${name})
  set_tests_properties(${name} PROPERTIES TIMEOUT 120)
endfunction()

if(NOT WIN32)
  rtxcov_add_test(test_status ${ACUDA_TESTS_DIR}/unit/test_status.c)
  rtxcov_add_test(test_memory ${ACUDA_TESTS_DIR}/integration/test_memory.c)
  rtxcov_add_test(test_stream_event ${ACUDA_TESTS_DIR}/integration/test_stream_event.c)
  rtxcov_add_test(test_ptx ${ACUDA_TESTS_DIR}/integration/test_ptx.c)
  target_compile_definitions(test_ptx PRIVATE ACUDA_TEST_PTX_DIR="${ACUDA_TESTS_DIR}/ptx")
  target_include_directories(test_ptx PRIVATE
    ${RTXDIASM_PTX_TO_SPIRV_DIR}/src
    ${RTXDIASM_PTX_TO_SPIRV_DIR}/src/include)
  if(UNIX)
    target_link_options(test_ptx PRIVATE -Wl,-z,stack-size=8388608)
  endif()

  rtxcov_add_test(test_driver ${ACUDA_TESTS_DIR}/integration/test_driver.c)
  target_link_libraries(test_driver PRIVATE dl)
  target_compile_definitions(test_driver PRIVATE
    ACUDA_DRIVER_PATH="$<TARGET_FILE:acuda_driver>"
    ACUDA_TEST_PTX_DIR="${ACUDA_TESTS_DIR}/ptx")

  rtxcov_add_test(test_texture_driver ${ACUDA_TESTS_DIR}/integration/test_texture_driver.c)
  target_link_libraries(test_texture_driver PRIVATE acuda_driver)
  target_compile_definitions(test_texture_driver PRIVATE ACUDA_TEST_PTX_DIR="${ACUDA_TESTS_DIR}/ptx")

  rtxcov_add_test(test_cudart ${ACUDA_TESTS_DIR}/integration/test_cudart.c)
  target_link_libraries(test_cudart PRIVATE acuda_cudart)
  target_compile_definitions(test_cudart PRIVATE ACUDA_TEST_PTX_DIR="${ACUDA_TESTS_DIR}/ptx")

  rtxcov_add_test(test_hostcall ${ACUDA_TESTS_DIR}/integration/test_hostcall.c)
  target_compile_definitions(test_hostcall PRIVATE ACUDA_TEST_PTX_DIR="${ACUDA_TESTS_DIR}/ptx")
  target_include_directories(test_hostcall PRIVATE ${RTX_MODULES_DIR}/rtxdiasm_acuda/src/runtime)

  rtxcov_add_test(test_ptx_kit ${ACUDA_TESTS_DIR}/integration/test_ptx_kit.c)
  target_compile_definitions(test_ptx_kit PRIVATE ACUDA_TEST_KIT_DIR="${ACUDA_TESTS_DIR}/windows-kit")
  add_test(NAME test_ptx_success_kit COMMAND test_ptx_kit "${ACUDA_TESTS_DIR}/windows-test-kit/ptx_to_spv_success/cases")
  set_tests_properties(test_ptx_success_kit PROPERTIES TIMEOUT 120)
  add_test(NAME test_ptx_error_kit COMMAND test_ptx_kit "${ACUDA_TESTS_DIR}/windows-test-kit/ptx_to_spv_error")
  set_tests_properties(test_ptx_error_kit PROPERTIES TIMEOUT 120)

  add_executable(bench_math ${ACUDA_TESTS_DIR}/bench/bench_math.c)
  target_link_libraries(bench_math PRIVATE acuda_runtime)
  target_compile_options(bench_math PRIVATE ${ACUDA_WARNINGS})
  add_executable(bench_gemm ${ACUDA_TESTS_DIR}/bench/bench_gemm.c)
  target_link_libraries(bench_gemm PRIVATE acuda_runtime)
  target_compile_options(bench_gemm PRIVATE ${ACUDA_WARNINGS})
  add_executable(bench_dp4a ${ACUDA_TESTS_DIR}/bench/bench_dp4a.c)
  target_link_libraries(bench_dp4a PRIVATE acuda_runtime)
  target_compile_options(bench_dp4a PRIVATE ${ACUDA_WARNINGS})
  target_compile_definitions(bench_dp4a PRIVATE ACUDA_BENCH_PTX_DIR="${ACUDA_TESTS_DIR}/bench")
  add_executable(bench_atomic ${ACUDA_TESTS_DIR}/bench/bench_atomic.c)
  target_link_libraries(bench_atomic PRIVATE acuda_runtime)
  target_compile_options(bench_atomic PRIVATE ${ACUDA_WARNINGS})
  if(NOT WIN32 AND DEFINED RTXCOV_TEST_DIR)
    foreach(_bench IN ITEMS bench_math bench_gemm bench_dp4a bench_atomic)
      set_target_properties(${_bench} PROPERTIES RUNTIME_OUTPUT_DIRECTORY ${RTXCOV_TEST_DIR})
      set_target_properties(${_bench} PROPERTIES
        RUNTIME_OUTPUT_DIRECTORY_RELEASE ${RTXCOV_OUTPUT_ROOT_RELEASE}/test
        RUNTIME_OUTPUT_DIRECTORY_DEBUG ${RTXCOV_OUTPUT_ROOT_DEBUG}/test
        RUNTIME_OUTPUT_DIRECTORY_RELWITHDEBINFO ${RTXCOV_OUTPUT_ROOT_RELEASE}/test
        RUNTIME_OUTPUT_DIRECTORY_MINSIZEREL ${RTXCOV_OUTPUT_ROOT_RELEASE}/test)
    endforeach()
    unset(_bench)
  endif()
else()
  # Windows test subset (portable/runtime API + driver/cudart smoke tests)
  set(_win_tests test_status test_memory test_stream_event test_ptx test_windows_smoke test_multi_device test_calchash_hang test_cudart)
  foreach(name IN ITEMS test_status test_memory test_stream_event)
    if(name STREQUAL "test_status")
      set(source ${ACUDA_TESTS_DIR}/unit/test_status.c)
    else()
      set(source ${ACUDA_TESTS_DIR}/integration/${name}.c)
    endif()
    add_executable(${name} ${source})
    target_link_libraries(${name} PRIVATE acuda_runtime)
    target_compile_options(${name} PRIVATE ${ACUDA_WARNINGS})
    add_test(NAME ${name} COMMAND ${name})
    set_tests_properties(${name} PROPERTIES TIMEOUT 180)
  endforeach()

  add_executable(test_ptx ${ACUDA_TESTS_DIR}/integration/test_ptx.c)
  target_link_libraries(test_ptx PRIVATE acuda_runtime)
  target_include_directories(test_ptx PRIVATE ${RTXDIASM_PTX_TO_SPIRV_DIR}/src ${RTXDIASM_PTX_TO_SPIRV_DIR}/src/include)
  target_compile_options(test_ptx PRIVATE ${ACUDA_WARNINGS})
  target_compile_definitions(test_ptx PRIVATE ACUDA_TEST_PTX_DIR="${ACUDA_TESTS_DIR}/ptx")
  if(MSVC)
    target_link_options(test_ptx PRIVATE /STACK:8388608)
  endif()
  add_test(NAME test_ptx COMMAND test_ptx)
  set_tests_properties(test_ptx PROPERTIES TIMEOUT 600)

  foreach(t IN ITEMS test_windows_smoke test_multi_device test_calchash_hang)
    add_executable(${t} ${ACUDA_TESTS_DIR}/integration/${t}.c)
    target_link_libraries(${t} PRIVATE acuda_driver)
    target_compile_options(${t} PRIVATE ${ACUDA_WARNINGS})
    if(t STREQUAL "test_windows_smoke")
      target_compile_definitions(${t} PRIVATE ACUDA_TEST_PTX_DIR="${ACUDA_TESTS_DIR}/ptx")
    endif()
    add_test(NAME ${t} COMMAND ${t})
    if(t STREQUAL "test_windows_smoke")
      set_tests_properties(${t} PROPERTIES TIMEOUT 180)
    elseif(t STREQUAL "test_calchash_hang")
      set_tests_properties(${t} PROPERTIES TIMEOUT 300)
    else()
      set_tests_properties(${t} PROPERTIES TIMEOUT 600)
    endif()
  endforeach()

  add_executable(test_cudart ${ACUDA_TESTS_DIR}/integration/test_cudart.c)
  target_link_libraries(test_cudart PRIVATE acuda_cudart)
  target_compile_options(test_cudart PRIVATE ${ACUDA_WARNINGS})
  target_compile_definitions(test_cudart PRIVATE ACUDA_TEST_PTX_DIR="${ACUDA_TESTS_DIR}/ptx")
  add_test(NAME test_cudart COMMAND test_cudart)
  set_tests_properties(test_cudart PROPERTIES TIMEOUT 300)

  if(DEFINED RTXCOV_TEST_DIR)
    foreach(_t IN LISTS _win_tests)
      if(TARGET ${_t})
        set_target_properties(${_t} PROPERTIES RUNTIME_OUTPUT_DIRECTORY ${RTXCOV_TEST_DIR})
        foreach(_cfg DEBUG RELEASE RELWITHDEBINFO MINSIZEREL)
          if(_cfg STREQUAL "DEBUG")
            set_target_properties(${_t} PROPERTIES RUNTIME_OUTPUT_DIRECTORY_${_cfg} ${RTXCOV_OUTPUT_ROOT_DEBUG}/test)
          else()
            set_target_properties(${_t} PROPERTIES RUNTIME_OUTPUT_DIRECTORY_${_cfg} ${RTXCOV_OUTPUT_ROOT_RELEASE}/test)
          endif()
        endforeach()
      endif()
    endforeach()
    unset(_t)
    unset(_cfg)
  endif()
  unset(_win_tests)
endif()
