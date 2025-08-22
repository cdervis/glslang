# Copyright (C) 2020-2025 The Khronos Group Inc.
#
# All rights reserved.
#
# Redistribution and use in source and binary forms, with or without
# modification, are permitted provided that the following conditions
# are met:
#
#    Redistributions of source code must retain the above copyright
#    notice, this list of conditions and the following disclaimer.
#
#    Redistributions in binary form must reproduce the above
#    copyright notice, this list of conditions and the following
#    disclaimer in the documentation and/or other materials provided
#    with the distribution.
#
#    Neither the name of The Khronos Group Inc. nor the names of its
#    contributors may be used to endorse or promote products derived
#    from this software without specific prior written permission.
#
# THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS
# "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT
# LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS
# FOR A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE
# COPYRIGHT HOLDERS OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT,
# INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING,
# BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES;
# LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER
# CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT
# LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN
# ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE
# POSSIBILITY OF SUCH DAMAGE.
cmake_minimum_required(VERSION 3.22.1)
project(glslang)

# Adhere to GNU filesystem layout conventions
include(CMakePackageConfigHelpers)

# Needed for cmake_dependent_option macro
include(CMakeDependentOption)

option(BUILD_EXTERNAL "Build external dependencies in /External" ON)
option(BUILD_WERROR "Enable warnings as errors (default is OFF)" OFF)

set(LIB_TYPE STATIC)

get_cmake_property(isMultiConfig "GENERATOR_IS_MULTI_CONFIG")
if (NOT isMultiConfig AND NOT CMAKE_BUILD_TYPE)
    # This logic inside SPIRV-Tools, which can upset build target dependencies
    # if changed after targets are already defined. To prevent these issues,
    # ensure CMAKE_BUILD_TYPE is assigned early and at the glslang root scope.
    message(STATUS "No build type selected, default to Debug")
    set(CMAKE_BUILD_TYPE "Debug")
endif()

option(ENABLE_SPIRV "Enables SPIRV output support" ON)
option(ENABLE_SPVREMAPPER "Enables building of SPVRemapper" ON)

option(ENABLE_GLSLANG_BINARIES "Builds glslang and spirv-remap" ON)

option(ENABLE_RTTI "Enables RTTI")
option(ENABLE_EXCEPTIONS "Enables Exceptions")
cmake_dependent_option(ENABLE_OPT "Enables spirv-opt capability if present" ON "ENABLE_SPIRV" OFF)

if(MINGW OR (APPLE AND ${CMAKE_CXX_COMPILER_ID} MATCHES "GNU"))
    # Workaround for CMake behavior on Mac OS with gcc, cmake generates -Xarch_* arguments
    # which gcc rejects
    set(ENABLE_PCH OFF)
    message(NOTICE "Disabling PCH")
endif()

option(ENABLE_PCH "Enables Precompiled header" ON)

if(ENABLE_SPIRV)
    add_compile_definitions(ENABLE_SPIRV)
endif()

if(WIN32)
    set(CMAKE_DEBUG_POSTFIX "d")
    add_compile_definitions(GLSLANG_OSINCLUDE_WIN32)
elseif(UNIX OR ANDROID)
    add_compile_definitions(GLSLANG_OSINCLUDE_UNIX)
else()
    message("unknown platform")
endif()

if(${CMAKE_CXX_COMPILER_ID} MATCHES "GNU")
    add_compile_options(-Wall -Wmaybe-uninitialized -Wuninitialized -Wunused -Wunused-local-typedefs -Wimplicit-fallthrough
                        -Wunused-parameter -Wunused-value  -Wunused-variable -Wunused-but-set-parameter -Wunused-but-set-variable)
    if(NOT ENABLE_RTTI)
        add_compile_options(-fno-rtti)
    endif()
    if(NOT ENABLE_EXCEPTIONS)
        add_compile_options(-fno-exceptions)
    endif()
    if(NOT CMAKE_CXX_COMPILER_VERSION VERSION_LESS "9.0.0")
        add_compile_options(-Werror=deprecated-copy)
    endif()

    if(NOT (CMAKE_SYSTEM_NAME STREQUAL "OpenBSD"))
        if (NOT APPLE)
            # Error if there's symbols that are not found at link time.
            add_link_options("-Wl,--no-undefined")
        endif()
    endif()
elseif(${CMAKE_CXX_COMPILER_ID} MATCHES "Clang" AND NOT MSVC)
    add_compile_options(-Wall -Wuninitialized -Wunused -Wunused-local-typedefs -Wimplicit-fallthrough
                        -Wunused-parameter -Wunused-value  -Wunused-variable)
    if(NOT ENABLE_RTTI)
        add_compile_options(-fno-rtti)
    endif()
    if(NOT ENABLE_EXCEPTIONS)
        add_compile_options(-fno-exceptions)
    endif()

    if(NOT (CMAKE_SYSTEM_NAME MATCHES "OpenBSD|Emscripten"))
        # Error if there's symbols that are not found at link time. Some linkers do not support this flag.
        if(NOT APPLE)
            add_link_options("-Wl,--no-undefined")
        endif()
    endif()
elseif(MSVC)
    if(NOT ENABLE_RTTI)
        add_compile_options(/GR-) # Disable RTTI
    endif()
    if(ENABLE_EXCEPTIONS)
        add_compile_options(/EHsc) # Enable Exceptions
	else()
        string(REGEX REPLACE "/EHsc" "" CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}") # Try to remove default /EHsc cxx_flag
        add_compile_options(/D_HAS_EXCEPTIONS=0)
    endif()
endif()

# NOTE we could potentially replace this logic with COMPILE_WARNING_AS_ERROR if cmake minimum is bumped to >= 3.24
if (BUILD_WERROR)
    if (NOT MSVC)
        add_compile_options(-Werror)
    else()
        add_compile_options(/WX)
    endif()
endif()

if(ENABLE_GLSLANG_JS)
    if(MSVC)
        add_compile_options(/Os /GR-)
    else()
        add_compile_options(-Os -fno-rtti -fno-exceptions)
        if(${CMAKE_CXX_COMPILER_ID} MATCHES "Clang" AND NOT MSVC)
            add_compile_options(-Wno-unused-parameter)
            add_compile_options(-Wno-unused-variable -Wno-unused-const-variable)
        endif()
    endif()
endif()

# Request C++17
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

function(glslang_set_link_args TARGET)
    # For MinGW compiles, statically link against the GCC and C++ runtimes.
    # This avoids the need to ship those runtimes as DLLs.
    # This is supported by GCC and Clang.
    if(MINGW)
        target_link_options(${TARGET} PRIVATE -static -static-libgcc -static-libstdc++)
    endif()
endfunction()

# Root directory for build-time generated include files
set(GLSLANG_GENERATED_INCLUDEDIR "${CMAKE_BINARY_DIR}/include")

################################################################################
# Build version information generation
################################################################################
include(parse_version.cmake)
set(GLSLANG_CHANGES_FILE      "${CMAKE_CURRENT_SOURCE_DIR}/CHANGES.md")
set(GLSLANG_BUILD_INFO_H_TMPL "${CMAKE_CURRENT_SOURCE_DIR}/build_info.h.tmpl")
set(GLSLANG_BUILD_INFO_H      "${GLSLANG_GENERATED_INCLUDEDIR}/glslang/build_info.h")

parse_version(${GLSLANG_CHANGES_FILE} GLSLANG)

function(configurate_version)
    set(major ${GLSLANG_VERSION_MAJOR})
    set(minor ${GLSLANG_VERSION_MINOR})
    set(patch ${GLSLANG_VERSION_PATCH})
    set(flavor ${GLSLANG_VERSION_FLAVOR})
    configure_file(${GLSLANG_BUILD_INFO_H_TMPL} ${GLSLANG_BUILD_INFO_H} @ONLY)
endfunction()

configurate_version()

# glslang_add_build_info_dependency() adds the glslang-build-info dependency and
# generated include directories to target.
function(glslang_add_build_info_dependency target)
    target_include_directories(${target} PUBLIC $<BUILD_INTERFACE:${GLSLANG_GENERATED_INCLUDEDIR}>)
endfunction()

# glslang_pch() adds precompiled header rules to <target> for the pre-compiled
# header file <pch>. As target_precompile_headers() was added in CMake 3.16,
# this is a no-op if called on earlier versions of CMake.
function(glslang_pch target pch)
    if(ENABLE_PCH)
        target_precompile_headers(${target} PRIVATE ${pch})
    endif()
endfunction()

if(ENABLE_OPT)
    message(STATUS "optimizer enabled")
    add_compile_definitions(ENABLE_OPT=1)
else()
    add_compile_definitions(ENABLE_OPT=0)
endif()

if(ENABLE_SPIRV)
    add_subdirectory(SPIRV)
endif()
add_subdirectory(glslang)

