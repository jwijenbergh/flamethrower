# vim: set ai ts=4 expandtab:
# ::=========================================================================::
# Build for flamethrower
# ::=========================================================================::

cmake_minimum_required(VERSION 3.5 FATAL_ERROR)

set(CMAKE_VERBOSE_MAKEFILE ON)

# The following must be set BEFORE doing project() or enable_language().
if (NOT CMAKE_BUILD_TYPE)
    message(STATUS "No build type defined; defaulting to 'Debug'")
    set(CMAKE_BUILD_TYPE "Debug" CACHE STRING
            "The type of build. Possible values are: Debug, Release, RelWithDebInfo and MinSizeRel.")
endif()
# ::-------------------------------------------------------------------------::

project(flamethrower VERSION 0.10.2)

list(APPEND CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake")

message(STATUS "The host system is: ${CMAKE_HOST_SYSTEM}.")
message(STATUS "Project base dir is: ${CMAKE_SOURCE_DIR}")

# Just for debugging when adding a new platform/compiler.  If you need to export
# compiler identity info into the project consider using: `include(WriteCompilerDetectionHeader)`
if (FLAME_BUILD_SHOW_CXX)
    message(STATUS "The detected C++ compiler supports these language features:")
    foreach(i ${CMAKE_CXX_COMPILE_FEATURES})
        message(STATUS "    ${i}")
    endforeach()
endif()

# Global for all targets
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
add_compile_options(-Wall)

# ::-------------------------------------------------------------------------::
# Find and setup all dependencies BEFORE declaring targets.
# ::-------------------------------------------------------------------------:: 

find_package(PkgConfig)

include(sanitizer)

pkg_search_module(LIBLDNS REQUIRED libldns ldns)
pkg_check_modules(LIBUV REQUIRED libuv>=1.23)
pkg_check_modules(LIBGNUTLS REQUIRED gnutls>=3.3)

option(DOH_ENABLE "Enable DNS over HTTPS (DoH) support" OFF)
if (DOH_ENABLE)
    pkg_search_module(LIBNGHTTP2 REQUIRED nghttp2 libnghttp2)
    message(STATUS "DNS over HTTPS (DoH) support is enabled")
    add_definitions(-DDOH_ENABLE)
else ()
    message(STATUS "DNS over HTTPS (DoH) support is disabled")
endif()

# ::-------------------------------------------------------------------------:: 
# BUILD TARGETS
# ::-------------------------------------------------------------------------:: 

set(flamecore_src
    flame/metrics.cpp
    flame/metrics.h
    flame/query.cpp
    flame/query.h
    flame/target.h
    flame/trafgen.cpp
    flame/trafgen.h
    flame/tcpsession.cpp
    flame/tcpsession.h
    flame/tcptlssession.cpp
    flame/tcptlssession.h
    flame/utils.cpp
    flame/utils.h
    )

set(flamecore_dirs
    PUBLIC ${LIBUV_INCLUDE_DIRS}
    PUBLIC ${LIBLDNS_INCLUDE_DIRS}
    PRIVATE ${LIBGNUTLS_INCLUDE_DIRS}
    PUBLIC "${CMAKE_SOURCE_DIR}/3rd/docopt.cpp"
    PUBLIC "${CMAKE_SOURCE_DIR}/3rd/json"
    PUBLIC "${CMAKE_SOURCE_DIR}/3rd/url-parser"
    PUBLIC "${CMAKE_SOURCE_DIR}/3rd/uvw"
    )

set(flamecore_libs
    PRIVATE ${LIBUV_LDFLAGS}
    PRIVATE ${LIBUV_LIBRARIES}
    PRIVATE ${LIBLDNS_LDFLAGS}
    PRIVATE ${LIBLDNS_LIBRARIES}
    PRIVATE ${LIBGNUTLS_LDFLAGS}
    PRIVATE ${LIBGNUTLS_LIBRARIES}
    )

set(flame_execs
    3rd/docopt.cpp/docopt.cpp
    3rd/url-parser/url_parser.c
    flame/main.cpp
    )

if (DOH_ENABLE)
    set(flamecore_src 
        ${flamecore_src}
        flame/http.h
        flame/httpssession.cpp
        flame/httpssession.h
        )

    set(flamecore_dirs
        ${flamecore_dirs}
        PRIVATE ${LIBNGHTTP2_INCLUDE_DIRS}
        PUBLIC "${CMAKE_SOURCE_DIR}/3rd/base64url"
        )

    set(flamecore_libs
        ${flamecore_libs}
        PRIVATE ${LIBNGHTTP2_LDFLAGS}
        PRIVATE ${LIBNGHTTP2_LIBRARIES}
        )

    set(flame_execs
        ${flame_execs}
        3rd/base64url/base64.cpp
        )
endif()

add_library(flamecore ${flamecore_src})

target_include_directories(flamecore ${flamecore_dirs})

target_link_libraries(flamecore ${flamecore_libs})

add_executable(flame ${flame_execs})

target_link_libraries(flame
        PRIVATE flamecore
        PRIVATE ${LIBUV_LDFLAGS}
        PRIVATE ${LIBUV_LIBRARIES}
        )

add_executable(tests
        tests/main.cpp
        )

target_include_directories(tests SYSTEM
        PRIVATE "${CMAKE_SOURCE_DIR}/3rd/catch"
        )

target_include_directories(tests
        PRIVATE "${CMAKE_SOURCE_DIR}/tests"
        PRIVATE "${CMAKE_SOURCE_DIR}/flame"
        )

target_link_libraries(tests
        PRIVATE flamecore
        )

install(TARGETS flame flamecore
        RUNTIME DESTINATION bin
        LIBRARY DESTINATION lib${LIB_SUFFIX}
        ARCHIVE DESTINATION lib/static)
