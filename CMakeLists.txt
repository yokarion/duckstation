cmake_minimum_required(VERSION 3.19)
project(duckstation C CXX)

# Policy settings.
cmake_policy(SET CMP0069 NEW)
set(CMAKE_POLICY_DEFAULT_CMP0069 NEW)

if(${CMAKE_SOURCE_DIR} STREQUAL ${CMAKE_BINARY_DIR})
  message(FATAL_ERROR "DuckStation does not support in-tree builds. Please make a build directory that is not the source"
                      "directory and generate your CMake project there using either `cmake -B build_directory` or by "
                      "running cmake from the build directory.")
endif()

if(NOT CMAKE_BUILD_TYPE MATCHES "Debug|Devel|MinSizeRel|RelWithDebInfo|Release")
  message(FATAL_ERROR "CMAKE_BUILD_TYPE not set. Please set it first.")
endif()

# Stop Windows builds
if(WIN32)
    message(FATAL_ERROR "Windows builds are disabled in this fork.")
endif()

# Pull in modules.
set(CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/CMakeModules/")
include(DuckStationUtils)

# Detect system attributes.
detect_operating_system()
detect_compiler()
detect_architecture()
detect_page_size()
detect_cache_line_size()

# Build options. Depends on system attributes.
include(DuckStationBuildOptions)
include(DuckStationDependencies)
include(DuckStationCompilerRequirement)

# Enable PIC on Linux, otherwise the builds do not support ASLR.
if(LINUX OR BSD)
  include(CheckPIESupported)
  check_pie_supported()
  set(CMAKE_POSITION_INDEPENDENT_CODE TRUE)
endif()

# Force debug symbols for Linux builds.
add_debug_symbol_flag(CMAKE_C_FLAGS_RELEASE)
add_debug_symbol_flag(CMAKE_CXX_FLAGS_RELEASE)

# Warning disables.
if(COMPILER_CLANG OR COMPILER_GCC)
  include(CheckCXXFlag)
  check_cxx_flag(-Wall COMPILER_SUPPORTS_WALL)
  check_cxx_flag(-Wno-class-memaccess COMPILER_SUPPORTS_MEMACCESS)
  check_cxx_flag(-Wno-invalid-offsetof COMPILER_SUPPORTS_OFFSETOF)
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wno-switch")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-switch")
endif()

# Disable exceptions and RTTI on non-MSVC compilers.
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fno-exceptions -fno-rtti")

# Write binaries to a separate directory.
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/bin")

# Installation directories. Everything goes into one directory.
if(ALLOW_INSTALL)
  set(CMAKE_INSTALL_BINDIR "${CMAKE_INSTALL_PREFIX}")
  set(CMAKE_INSTALL_LIBDIR "${CMAKE_INSTALL_PREFIX}")
endif()

# Enable large file support on Linux 32-bit platforms.
if(CMAKE_SIZEOF_VOID_P EQUAL 4)
  add_definitions("-D_FILE_OFFSET_BITS=64")
endif()

# Optional unit tests.
if(BUILD_TESTS)
  enable_testing()
endif()

# Prevent fmt from being built with exceptions.
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DFMT_EXCEPTIONS=0")

# Recursively include the source tree.
add_subdirectory(dep)
add_subdirectory(src)

# Output build summary.
include(DuckStationBuildSummary)
