# SPDX-License-Identifier: LGPL-2.1-or-later
# Optional NDK build adapter maintained on the tencentmalos integration branch.
cmake_minimum_required(VERSION 3.22.1)
if(CMAKE_SOURCE_DIR STREQUAL CMAKE_CURRENT_SOURCE_DIR)
    project(ffmpeg_android LANGUAGES C CXX)
endif()
if(NOT ANDROID OR NOT ANDROID_ABI STREQUAL "arm64-v8a")
    message(FATAL_ERROR "FFmpeg Android adapter currently validates arm64-v8a with the NDK toolchain")
endif()
include(ExternalProject)
find_program(FFMPEG_MAKE NAMES gmake make REQUIRED)
get_filename_component(ffmpeg_source "${CMAKE_CURRENT_LIST_DIR}/../.." ABSOLUTE)
get_filename_component(tool_bin "${CMAKE_C_COMPILER}" DIRECTORY)
# The NDK legacy toolchain sets CMAKE_SYSTEM_VERSION=1; use its resolved API.
if(DEFINED ANDROID_PLATFORM_LEVEL)
    set(api "${ANDROID_PLATFORM_LEVEL}")
else()
    set(api "${CMAKE_SYSTEM_VERSION}")
endif()
set(cc "${tool_bin}/aarch64-linux-android${api}-clang")
set(cxx "${tool_bin}/aarch64-linux-android${api}-clang++")
if(NOT EXISTS "${cc}")
    message(FATAL_ERROR "No NDK compiler wrapper for requested API ${api}")
endif()
set(FFMPEG_ANDROID_JOBS "4" CACHE STRING "FFmpeg make workers")
if(NOT FFMPEG_ANDROID_JOBS MATCHES "^[1-9][0-9]*$")
    message(FATAL_ERROR "FFMPEG_ANDROID_JOBS must be positive")
endif()
option(FFMPEG_ANDROID_NETWORK "Enable FFmpeg network protocols" OFF)
set(FFMPEG_ANDROID_DECODERS "" CACHE STRING "Optional decoder allowlist; empty keeps built-in decoders")
set(FFMPEG_ANDROID_DEMUXERS "" CACHE STRING "Optional demuxer allowlist; empty keeps built-in demuxers")
set(prefix "${CMAKE_CURRENT_BINARY_DIR}/prefix")
set(options --disable-autodetect --enable-cross-compile --target-os=android --arch=aarch64
    --enable-static --disable-shared --enable-pic --disable-programs --disable-doc --disable-debug
    --disable-avdevice --disable-postproc --disable-encoders --disable-muxers
    --disable-hwaccels --disable-vulkan --disable-mediacodec --disable-jni --enable-zlib)
if(NOT FFMPEG_ANDROID_NETWORK)
    list(APPEND options --disable-network)
endif()
if(FFMPEG_ANDROID_DECODERS)
    list(JOIN FFMPEG_ANDROID_DECODERS "," decoders)
    list(APPEND options --disable-decoders "--enable-decoder=${decoders}")
endif()
if(FFMPEG_ANDROID_DEMUXERS)
    list(JOIN FFMPEG_ANDROID_DEMUXERS "," demuxers)
    list(APPEND options --disable-demuxers "--enable-demuxer=${demuxers}")
endif()
set(libraries avformat avcodec avfilter swscale swresample avutil)
set(byproducts)
foreach(lib IN LISTS libraries)
    list(APPEND byproducts "${prefix}/lib/lib${lib}.a")
endforeach()
# Include path must exist when the importing CMake target is generated.
file(MAKE_DIRECTORY "${prefix}/include")
ExternalProject_Add(ffmpeg_android_build
    SOURCE_DIR "${ffmpeg_source}"
    BINARY_DIR "${CMAKE_CURRENT_BINARY_DIR}/build"
    INSTALL_DIR "${prefix}"
    DOWNLOAD_COMMAND ""
    UPDATE_COMMAND ""
    CONFIGURE_COMMAND "${ffmpeg_source}/configure" "--prefix=<INSTALL_DIR>"
        "--cc=${cc}" "--cxx=${cxx}" "--ar=${tool_bin}/llvm-ar"
        "--ranlib=${tool_bin}/llvm-ranlib" "--nm=${tool_bin}/llvm-nm" "--strip=${tool_bin}/llvm-strip"
        "--sysroot=${CMAKE_SYSROOT}" --pkg-config=false ${options}
    # Install in the same step that owns BUILD_BYPRODUCTS. With CMake 3.22,
    # declaring installed archives as build outputs but copying them only in a
    # later install step lets Ninja restat skip a required consumer relink.
    BUILD_COMMAND "${FFMPEG_MAKE}" "-j${FFMPEG_ANDROID_JOBS}"
        COMMAND "${FFMPEG_MAKE}" install
    INSTALL_COMMAND ""
    BUILD_BYPRODUCTS ${byproducts}
    BUILD_ALWAYS TRUE
    LOG_CONFIGURE ON LOG_BUILD ON LOG_INSTALL ON
    LOG_OUTPUT_ON_FAILURE ON)
add_library(ffmpeg_android INTERFACE)
add_library(FFmpeg::ffmpeg ALIAS ffmpeg_android)
foreach(lib IN LISTS libraries)
    add_library(ffmpeg_android_${lib} STATIC IMPORTED GLOBAL)
    set_target_properties(ffmpeg_android_${lib} PROPERTIES
        IMPORTED_LOCATION "${prefix}/lib/lib${lib}.a"
        INTERFACE_INCLUDE_DIRECTORIES "${prefix}/include")
    add_library(FFmpeg::${lib} ALIAS ffmpeg_android_${lib})
    add_dependencies(ffmpeg_android_${lib} ffmpeg_android_build)
endforeach()
# Explicit cyclic archive group, then NDK-provided dependencies. No host pkg-config.
target_link_libraries(ffmpeg_android INTERFACE -Wl,--start-group
    ffmpeg_android_avformat ffmpeg_android_avcodec ffmpeg_android_avfilter
    ffmpeg_android_swscale ffmpeg_android_swresample ffmpeg_android_avutil
    -Wl,--end-group z m dl)
# These archives are private to the consuming DSO. Bind internal AArch64 NEON
# table references locally and do not export/interpose another FFmpeg instance.
# --enable-pic alone does not make those assembly references preemptible.
foreach(lib IN LISTS libraries)
    target_link_options(ffmpeg_android INTERFACE "-Wl,--exclude-libs,lib${lib}.a")
endforeach()
add_dependencies(ffmpeg_android ffmpeg_android_build)
