cmake_minimum_required(VERSION 3.16)

project(x264 VERSION 0.165 LANGUAGES C)

include(GNUInstallDirs)

option(X264_BUILD_CLI "Build the x264 command-line encoder" ON)
option(X264_ENABLE_ASM "Enable platform-specific assembly optimizations when supported" ON)
option(X264_ENABLE_THREAD "Enable multithreaded encoding" ON)
option(X264_ENABLE_OPENCL "Enable OpenCL support" OFF)
option(X264_ENABLE_GPL "Enable GPL-only features" ON)
set(X264_TARGET_ARCH "auto" CACHE STRING "Target architecture (auto, x86, x86_64, arm, or aarch64)")
set_property(CACHE X264_TARGET_ARCH PROPERTY STRINGS auto x86 x86_64 arm aarch64)
set(X264_BIT_DEPTH "8" CACHE STRING "Output bit depth (8, 10, or all)")
set_property(CACHE X264_BIT_DEPTH PROPERTY STRINGS 8 10 all)
set(X264_CHROMA_FORMAT "all" CACHE STRING "Output chroma format (400, 420, 422, 444, or all)")
set_property(CACHE X264_CHROMA_FORMAT PROPERTY STRINGS 400 420 422 444 all)

if(NOT X264_BIT_DEPTH MATCHES "^(8|10|all)$")
    message(FATAL_ERROR "X264_BIT_DEPTH must be 8, 10, or all")
endif()
if(NOT X264_CHROMA_FORMAT MATCHES "^(400|420|422|444|all)$")
    message(FATAL_ERROR "X264_CHROMA_FORMAT must be 400, 420, 422, or all")
endif()

if(X264_TARGET_ARCH STREQUAL "auto")
    set(X264_TARGET_ARCH "${CMAKE_SYSTEM_PROCESSOR}")
    if(NOT X264_TARGET_ARCH)
        set(X264_TARGET_ARCH "${CMAKE_HOST_SYSTEM_PROCESSOR}")
    endif()
    if(NOT X264_TARGET_ARCH AND WIN32 AND CMAKE_SIZEOF_VOID_P EQUAL 8)
        set(X264_TARGET_ARCH x86_64)
    endif()
endif()

if(X264_BIT_DEPTH STREQUAL "all")
    set(X264_CONFIG_BIT_DEPTH 0)
else()
    set(X264_CONFIG_BIT_DEPTH ${X264_BIT_DEPTH})
endif()
if(X264_CHROMA_FORMAT STREQUAL "all")
    set(X264_CONFIG_CHROMA_FORMAT 0)
else()
    set(X264_CONFIG_CHROMA_FORMAT X264_CSP_I${X264_CHROMA_FORMAT})
endif()

set(X264_COMMON_SOURCES
    common/osdep.c common/base.c common/cpu.c common/tables.c
    encoder/api.c
    common/mc.c common/predict.c common/pixel.c common/macroblock.c
    common/frame.c common/dct.c common/cabac.c common/common.c
    common/rectangle.c common/set.c common/quant.c common/deblock.c
    common/vlc.c common/mvpred.c common/bitstream.c
    encoder/analyse.c encoder/me.c encoder/ratecontrol.c encoder/set.c
    encoder/macroblock.c encoder/cabac.c encoder/cavlc.c encoder/encoder.c
    encoder/lookahead.c
)

set(X264_CLI_SOURCES
    x264.c autocomplete.c input/input.c input/timecode.c input/raw.c
    input/y4m.c output/raw.c output/matroska.c output/matroska_ebml.c
    output/flv.c output/flv_bytestream.c filters/filters.c
    filters/video/video.c filters/video/source.c filters/video/internal.c
    filters/video/resize.c filters/video/fix_vfr_pts.c
    filters/video/select_every.c filters/video/crop.c
)
set(HAVE_GETOPT_LONG 1)
if(WIN32)
    set(HAVE_GETOPT_LONG 0)
    list(APPEND X264_CLI_SOURCES extras/getopt.c)
endif()

set(X264_ASM_SOURCES)
set(X264_ASM_ENABLED FALSE)
if(X264_ENABLE_ASM AND X264_TARGET_ARCH MATCHES "^(x86_64|AMD64|amd64|i.86)$")
    find_program(X264_NASM_EXECUTABLE NAMES nasm yasm)
    if(X264_NASM_EXECUTABLE)
        enable_language(ASM_NASM)
        set(X264_ASM_ENABLED TRUE)
        list(APPEND X264_ASM_SOURCES
            common/x86/cpu-a.asm common/x86/bitstream-a.asm
            common/x86/const-a.asm common/x86/cabac-a.asm
            common/x86/dct-a.asm common/x86/deblock-a.asm
            common/x86/mc-a.asm common/x86/mc-a2.asm
            common/x86/pixel-a.asm common/x86/predict-a.asm
            common/x86/quant-a.asm common/x86/dct-64.asm
            common/x86/trellis-64.asm common/x86/sad-a.asm
        )
        list(APPEND X264_COMMON_SOURCES common/x86/mc-c.c common/x86/predict-c.c)
    else()
        message(WARNING "X264_ENABLE_ASM is ON but NASM/YASM was not found; using C implementations")
    endif()
endif()

set(HAVE_THREAD 0)
set(HAVE_WIN32THREAD 0)
set(HAVE_POSIXTHREAD 0)
if(X264_ENABLE_THREAD)
    find_package(Threads QUIET)
    if(Threads_FOUND)
        set(HAVE_THREAD 1)
        if(MSVC)
            set(HAVE_WIN32THREAD 1)
        else()
            set(HAVE_POSIXTHREAD 1)
            list(APPEND X264_COMMON_SOURCES common/threadpool.c)
        endif()
        list(APPEND X264_COMMON_SOURCES common/threadpool.c)
        if(HAVE_WIN32THREAD)
            list(APPEND X264_COMMON_SOURCES common/win32thread.c)
        endif()
    else()
        message(WARNING "Threads were requested but no thread library was found; disabling threads")
    endif()
endif()

set(HAVE_OPENCL 0)
if(X264_ENABLE_OPENCL)
    find_package(OpenCL QUIET)
    if(OpenCL_FOUND)
        set(HAVE_OPENCL "(BIT_DEPTH==8)")
        list(APPEND X264_COMMON_SOURCES common/opencl.c encoder/slicetype-cl.c)
    else()
        message(WARNING "OpenCL was requested but not found; disabling OpenCL")
    endif()
endif()

if(X264_ENABLE_GPL)
    set(HAVE_GPL 1)
else()
    set(HAVE_GPL 0)
endif()
set(HAVE_INTERLACED 1)
set(HAVE_MMX 0)
set(HAVE_SSE 0)
set(HAVE_AVX 0)
if(X264_ASM_ENABLED)
    set(HAVE_MMX 1)
    set(HAVE_SSE 1)
endif()

set(X264_CONFIG_DIR "${CMAKE_CURRENT_BINARY_DIR}/generated")
configure_file(cmake/config.h.in "${X264_CONFIG_DIR}/config.h")
configure_file(cmake/x264_config.h.in "${X264_CONFIG_DIR}/x264_config.h")

add_library(x264 ${X264_COMMON_SOURCES} ${X264_ASM_SOURCES})
add_library(x264::x264 ALIAS x264)
set_target_properties(x264 PROPERTIES
    OUTPUT_NAME x264
    C_STANDARD 99
    C_STANDARD_REQUIRED ON
    POSITION_INDEPENDENT_CODE ON
    PUBLIC_HEADER "${CMAKE_CURRENT_SOURCE_DIR}/x264.h;${X264_CONFIG_DIR}/x264_config.h")

target_include_directories(x264
    PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
        $<BUILD_INTERFACE:${X264_CONFIG_DIR}>
        $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
    PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/common ${CMAKE_CURRENT_SOURCE_DIR}/common/x86)
target_compile_definitions(x264 PRIVATE HAVE_CONFIG_H=1 BIT_DEPTH=8 HIGH_BIT_DEPTH=0)
if(X264_ASM_ENABLED)
    set_source_files_properties(${X264_ASM_SOURCES} PROPERTIES
    COMPILE_DEFINITIONS "BIT_DEPTH=8;private_prefix=x264_8;ARCH_X86_64=1")
endif()
if(HAVE_THREAD)
    target_link_libraries(x264 PRIVATE Threads::Threads)
endif()
if(OpenCL_FOUND)
    target_link_libraries(x264 PRIVATE OpenCL::OpenCL)
endif()

message(STATUS "x264 assembly: ${X264_ASM_ENABLED}")

if(X264_BUILD_CLI)
    add_executable(x264-cli ${X264_CLI_SOURCES})
    set_target_properties(x264-cli PROPERTIES OUTPUT_NAME x264 C_STANDARD 99 C_STANDARD_REQUIRED ON)
    target_include_directories(x264-cli PRIVATE ${CMAKE_CURRENT_SOURCE_DIR} ${CMAKE_CURRENT_SOURCE_DIR}/extras ${X264_CONFIG_DIR})
    target_compile_definitions(x264-cli PRIVATE HAVE_CONFIG_H=1 BIT_DEPTH=8 HIGH_BIT_DEPTH=0)
    target_link_libraries(x264-cli PRIVATE x264)
endif()

include(CMakePackageConfigHelpers)
write_basic_package_version_file(
    "${CMAKE_CURRENT_BINARY_DIR}/x264ConfigVersion.cmake"
    VERSION ${PROJECT_VERSION}
    COMPATIBILITY SameMajorVersion)

install(TARGETS x264
    EXPORT x264Targets
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})
if(X264_BUILD_CLI)
    install(TARGETS x264-cli RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})
endif()
install(EXPORT x264Targets
    FILE x264Targets.cmake
    NAMESPACE x264::
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/x264)
install(FILES
    "${CMAKE_CURRENT_BINARY_DIR}/x264ConfigVersion.cmake"
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/x264)

message(STATUS "x264 bit depth: ${X264_BIT_DEPTH}")
message(STATUS "x264 CLI: ${X264_BUILD_CLI}")
message(STATUS "x264 threads: ${HAVE_THREAD}")
message(STATUS "x264 OpenCL: ${HAVE_OPENCL}")