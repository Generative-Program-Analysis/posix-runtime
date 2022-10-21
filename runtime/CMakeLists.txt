#===------------------------------------------------------------------------===#
#
#                     The KLEE Symbolic Virtual Machine
#
# This file is distributed under the University of Illinois Open Source
# License. See LICENSE.TXT for details.
#
#===------------------------------------------------------------------------===#

# Handle bitcode libraries
# Define the different configurations to be compiled and made available using a specific suffix
set(bc_architectures 64)

# Todo 32bit support

set(available_engine_types
  "klee"
  "gensym"
)

set(LIB_BC_PREFIX ${available_engine_types})

set(LIB_BC_SUFFIX ${bc_architectures})

if (NOT DEFINED RUNTIME_CFLAGS)
  message(FATAL_ERROR "Please provide runtime library compilation flag with by passing `-DRUNTIME_CFLAGS=\"flags\"` to CMake.")
endif()

foreach (arch_ty ${bc_architectures})
  foreach (engine_ty ${available_engine_types})

    # Set specific flags for that configuration
    set(local_flags "")

    # Set specific compiler flags depending on the optimization
    if (engine_ty STREQUAL "klee")
        list(APPEND local_flags "-D__USE_KLEE__")
    else()
        list(APPEND local_flags "-U__USE_KLEE__")
    endif ()

    # Define suffix-specific optimizations
    set("${engine_ty}_LIB_BC_FLAGS_${arch_ty}" ${local_flags})
  endforeach ()
endforeach ()

message(STATUS "KLEE_RUNTIME_DIRECTORY: ${KLEE_RUNTIME_DIRECTORY}")

# Common for all library configurations
# Since the runtime now contains fortified libc functions, it is
# important to compile it with -D_FORTIFY_SOURCE=0 to avoid infinite
# recursion
set(COMMON_CC_FLAGS
        "-I${CMAKE_SOURCE_DIR}/include"
        "-I${CMAKE_BINARY_DIR}/include"
        -D_FORTIFY_SOURCE=0
        -D_GNU_SOURCE
        -D__STDC_LIMIT_MACROS
        -D__STDC_CONSTANT_MACROS
        -Wall
        -Wwrite-strings
        -Xclang
        -disable-O0-optnone
        )

string(REPLACE " " ";" _compile_flags ${RUNTIME_CFLAGS})
foreach (_prefix ${LIB_BC_PREFIX})
  foreach (_suffix ${LIB_BC_SUFFIX})
    list(APPEND "${_prefix}_LIB_BC_FLAGS_${_suffix}" ${_compile_flags})
    list(APPEND "${_prefix}_LIB_BC_FLAGS_${_suffix}" ${COMMON_CC_FLAGS})
    message(STATUS "${_prefix}_LIB_BC_FLAGS_${_suffix}: ${${_prefix}_LIB_BC_FLAGS_${_suffix}}")
  endforeach ()
endforeach ()
unset(_compile_flags)

set(RUNTIME_LIBRARIES RuntimePOSIX)

add_subdirectory(POSIX)

add_custom_target(BuildPOSIXRuntimes
        ALL
        DEPENDS "${RUNTIME_LIBRARIES}"
        )

install(DIRECTORY "${KLEE_RUNTIME_DIRECTORY}/"
        DESTINATION "${KLEE_INSTALL_RUNTIME_DIR}"
        )
