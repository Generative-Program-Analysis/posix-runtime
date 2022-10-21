#===------------------------------------------------------------------------===#
#
#                     The KLEE Symbolic Virtual Machine
#
# This file is distributed under the University of Illinois Open Source
# License. See LICENSE.TXT for details.
#
#===------------------------------------------------------------------------===#

###############################################################################
# Minimum CMake version and policies
###############################################################################
cmake_minimum_required(VERSION 3.9.0)
if (POLICY CMP0054)
  cmake_policy(SET CMP0054 NEW)
endif()

if (POLICY CMP0042)
  # Enable `MACOSX_RPATH` by default.
  cmake_policy(SET CMP0042 NEW)
endif()

if (POLICY CMP0037)
  # Disallow reserved target names
  cmake_policy(SET CMP0037 NEW)
endif()

project(KLEE-POSIX CXX C)

include(CheckFunctionExists)
include(CheckLibraryExists)

###############################################################################
# Project version
###############################################################################
set(KLEE_VERSION_MAJOR 2)
set(KLEE_VERSION_MINOR 3)
set(KLEE_VERSION "${KLEE_VERSION_MAJOR}.${KLEE_VERSION_MINOR}")

# If a patch is needed, we can add KLEE_VERSION_PATCH
# set(KLEE_VERSION_PATCH 0)
# set(KLEE_VERSION "${KLEE_VERSION_MAJOR}.${KLEE_VERSION_MINOR}.${KLEE_VERSION_PATCH}")

message(STATUS "KLEE version ${KLEE_VERSION}")
set(PACKAGE_STRING "\"KLEE ${KLEE_VERSION}\"")
set(PACKAGE_URL "\"https://klee.github.io\"")

################################################################################
# Sanity check - Disallow building in source.
################################################################################
if ("${CMAKE_SOURCE_DIR}" STREQUAL "${CMAKE_BINARY_DIR}")
  message(FATAL_ERROR "In source builds are not allowed. You should invoke "
          "CMake from a different directory.")
endif()

# Reference specific library paths used during linking for install
SET(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)

################################################################################
# Include useful CMake functions
################################################################################
include(GNUInstallDirs)
include(CheckCXXSymbolExists)
include(CheckIncludeFile)
include(CheckIncludeFileCXX)
include(CheckPrototypeDefinition)
include(CMakePushCheckState)
include("${CMAKE_SOURCE_DIR}/cmake/string_to_list.cmake")


################################################################################
# Find LLVM
################################################################################
include(${CMAKE_SOURCE_DIR}/cmake/find_llvm.cmake)
set(NEEDED_LLVM_VARS
  LLVM_PACKAGE_VERSION
  LLVM_VERSION_MAJOR
  LLVM_VERSION_MINOR
  LLVM_VERSION_PATCH
  LLVM_DEFINITIONS
  LLVM_ENABLE_ASSERTIONS
  LLVM_ENABLE_EH
  LLVM_ENABLE_RTTI
  LLVM_INCLUDE_DIRS
  LLVM_LIBRARY_DIRS
  LLVM_TOOLS_BINARY_DIR
  LLVM_ENABLE_VISIBILITY_INLINES_HIDDEN
  TARGET_TRIPLE
)

if (${LLVM_VERSION_MAJOR} LESS 9)
  message(FATAL_ERROR "only support for LLVM >= 9.0.0")
endif()

foreach (vname ${NEEDED_LLVM_VARS})
  message(STATUS "${vname}: \"${${vname}}\"")
  if (NOT (DEFINED "${vname}"))
    message(FATAL_ERROR "${vname} was not defined")
  endif()
endforeach()

set(OPTIONAL_LLVM_VARS
  LLVM_BUILD_MAIN_SRC_DIR
)
foreach (vname ${OPTIONAL_LLVM_VARS})
  if (${vname})
    message(STATUS "${vname}: \"${${vname}}\"")
  endif()
endforeach()

# Find llvm-link
set(LLVM_LINK "${LLVM_TOOLS_BINARY_DIR}/llvm-link")
if (NOT EXISTS "${LLVM_LINK}")
  message(FATAL_ERROR "Failed to find llvm-link at \"${LLVM_LINK}\"")
endif()

# Find llvm-ar
set(LLVM_AR "${LLVM_TOOLS_BINARY_DIR}/llvm-ar")
if (NOT EXISTS "${LLVM_AR}")
  message(FATAL_ERROR "Failed to find llvm-ar at \"${LLVM_AR}\"")
endif()

# Find llvm-as
set(LLVM_AS "${LLVM_TOOLS_BINARY_DIR}/llvm-as")
if (NOT EXISTS "${LLVM_AS}")
  message(FATAL_ERROR "Failed to find llvm-as at \"${LLVM_AS}\"")
endif()

################################################################################
# Find bitcode compiler
################################################################################
include("${CMAKE_SOURCE_DIR}/cmake/find_bitcode_compiler.cmake")
message(STATUS "LLVMCC: ${LLVMCC}")
if (NOT EXISTS "${LLVMCC}")
  message(FATAL_ERROR "Cannot find C bitcode compiler \"${LLVMCC}\"")
endif()
message(STATUS "LLVMCXX: ${LLVMCXX}")
if (NOT EXISTS "${LLVMCXX}")
  message(FATAL_ERROR "Cannot find C++ bitcode compiler \"${LLVMCXX}\"")
endif()

################################################################################
# C++ version
################################################################################
set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

################################################################################
# Miscellaneous header file detection
################################################################################
check_include_file(sys/statfs.h HAVE_SYSSTATFS_H)

# FIXME: This is needed by the runtime not KLEE itself so we are testing the wrong
# compiler.
check_include_file("selinux/selinux.h" HAVE_SELINUX_SELINUX_H)
check_include_file("sys/acl.h" HAVE_SYS_ACL_H)
if (HAVE_SELINUX_SELINUX_H)
  message(STATUS "SELinux support enabled")
  set(HAVE_SELINUX 1)
  # Test what function signature we need to use for SELinux. The signatures
  # have changed between 2.2 and 2.3. In particular, the type of the "security
  # context" parameter was changed from char * to const char *, with this
  # patch: [PATCH] Get rid of security_context_t and fix const declarations.
  # [http://www.spinics.net/lists/selinux/msg14827.html]
  check_prototype_definition(setcon
    "int setcon(char* context)"
    "0"
    "selinux/selinux.h"
    SELINUX_SECURITY_CTX_NO_CONST
  )
  if (SELINUX_SECURITY_CTX_NO_CONST)
    message(STATUS "security_context_t is char*")
    set(KLEE_SELINUX_CTX_CONST " ")
  else()
    check_prototype_definition(setcon
      "int setcon(const char* context)"
      "0"
      "selinux/selinux.h"
      SELINUX_SECURITY_CTX_WITH_CONST
    )
    if (SELINUX_SECURITY_CTX_WITH_CONST)
      message(STATUS "security_context_t is const char*")
      set(KLEE_SELINUX_CTX_CONST "const")
    else()
      message(FATAL_ERROR "Failed to determine function signature for \"setcon\"")
    endif()
  endif()
else()
  message(STATUS "SELinux support disabled")
  set(HAVE_SELINUX 0)
endif()

################################################################################
# KLEE runtime support
################################################################################
# This is set here and not in `runtime` because `config.h` needs to be generated.

# Todo: Check availability of 32bit support for platform

set(KLEE_INSTALL_RUNTIME_DIR "${CMAKE_INSTALL_FULL_LIBDIR}")

# Location where KLEE will look for the built runtimes by default.
set(KLEE_RUNTIME_DIRECTORY "${CMAKE_BINARY_DIR}/runtime/lib")

file(MAKE_DIRECTORY ${KLEE_RUNTIME_DIRECTORY})

################################################################################
# GENSYM and KLEE external header path
################################################################################
#set(ABSOLUTE_GENSYM_H_PATH "")
#set(ABSOLUTE_KLEE_H_PATH "")
if (NOT DEFINED GENSYM_HEADER_DIR)
  message(FATAL_ERROR "Please provide gensym header path by passing `-DGENSYM_HEADER_DIR=\"directory-to-your-gensym_client.h\"` to CMake.")
endif()
get_filename_component(ABSOLUTE_GENSYM_H_PATH ${GENSYM_HEADER_DIR} ABSOLUTE)
if (NOT DEFINED KLEE_HEADER_DIR)
  message(FATAL_ERROR "Please provide klee header path by passing `-DKLEE_HEADER_DIR=\"directory-to-your-klee.h\"` to CMake.")
endif()
get_filename_component(ABSOLUTE_KLEE_H_PATH ${KLEE_HEADER_DIR} ABSOLUTE)
set(ABSOLUTE_GENSYM_H_PATH "${ABSOLUTE_GENSYM_H_PATH}/gensym_client.h")
set(ABSOLUTE_KLEE_H_PATH "${ABSOLUTE_KLEE_H_PATH}/klee.h")
message(STATUS "ABSOLUTE_GENSYM_H_PATH : ${ABSOLUTE_GENSYM_H_PATH}")
message(STATUS "ABSOLUTE_KLEE_H_PATH : ${ABSOLUTE_KLEE_H_PATH}")
check_include_file("${ABSOLUTE_GENSYM_H_PATH}" HAVE_GENSYM_H)
check_include_file("${ABSOLUTE_KLEE_H_PATH}" HAVE_KLEE_H)
if (NOT (HAVE_GENSYM_H AND HAVE_KLEE_H))
  message(FATAL_ERROR "Can not find gensym_client.h and klee.h")
endif()
set(ABSOLUTE_GENSYM_H_PATH "\"${ABSOLUTE_GENSYM_H_PATH}\"")
set(ABSOLUTE_KLEE_H_PATH "\"${ABSOLUTE_KLEE_H_PATH}\"")

################################################################################
# Generate `engine_api.h`
################################################################################
configure_file(${CMAKE_SOURCE_DIR}/include/posix-runtime/engine_api.h.cmin
  ${CMAKE_BINARY_DIR}/include/posix-runtime/engine_api.h)

################################################################################
# Generate `config.h`
################################################################################
configure_file(${CMAKE_SOURCE_DIR}/include/posix-runtime/Config/config.h.cmin
  ${CMAKE_BINARY_DIR}/include/posix-runtime/Config/config.h)

################################################################################
# Global include directories
################################################################################
include_directories("${CMAKE_BINARY_DIR}/include")
include_directories("${CMAKE_SOURCE_DIR}/include")


################################################################################
# Set default location for targets in the build directory
################################################################################
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)

################################################################################
# KLEE runtime
################################################################################
add_subdirectory(runtime)