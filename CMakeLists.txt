# Copyright © 2020, Intel Corporation
#
# Permission is hereby granted, free of charge, to any person obtaining a
# copy of this software and associated documentation files (the "Software"),
# to deal in the Software without restriction, including without limitation
# the rights to use, copy, modify, merge, publish, distribute, sublicense,
# and/or sell copies of the Software, and to permit persons to whom the
# Software is furnished to do so, subject to the following conditions:
#
# The above copyright notice and this permission notice shall be included
# in all copies or substantial portions of the Software.
#
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
# IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL
# THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING
# FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS
# IN THE SOFTWARE.
cmake_minimum_required (VERSION 2.6)

#################################################################################
# CMAKE POLICIES
#################################################################################
# Cmake policies based on instr.cmake
if (COMMAND cmake_policy)
    if (POLICY CMP0005)
        # automatic quoting of brackets
        cmake_policy (SET CMP0005 NEW)
    endif (POLICY CMP0005)

    if (POLICY CMP0015)
        # .vcproj contains relative paths to additional library directories, thus we need to be able to cope with that
        cmake_policy (SET CMP0015 NEW)
    endif (POLICY CMP0015)
endif (COMMAND cmake_policy)

# Uncomment for logs during build (during 'make' comamnd)
#set (CMAKE_VERBOSE_MAKEFILE ON)

#################################################################################
# TARGET
#################################################################################

set (PROJECT_NAME metrics_discovery)
project (${PROJECT_NAME})

set (LIB md)
set (LIB_NAME libmd)

include(md_cmake_adapter.cmake)

#################################################################################
# INCLUDES
#################################################################################

# generic
include_directories (
    # external
    ${BS_DIR_EXTERNAL}/mesa/include
    ${BS_DIR_EXTERNAL}/mesa/src
    ${BS_DIR_EXTERNAL}/mesa/src/util
    ${BS_DIR_EXTERNAL}/mesa/src/intel/dev
    ${BS_DIR_EXTERNAL}/drm
    # mdapi core
    ${BS_DIR_INSTRUMENTATION}/metrics_discovery/linux/inc
    ${BS_DIR_INSTRUMENTATION}/metrics_discovery/common/inc
    # utils
    ${BS_DIR_INSTRUMENTATION}/utils/common/inc
    ${BS_DIR_INSTRUMENTATION}/utils/linux/inc
    # common
    ${BS_DIR_INC}/common
    ${BS_DIR_INC}/common/instrumentation
    ${BS_DIR_INC}/common/instrumentation/api
)

# drm
if (NOT "${LIBDRM_SRC}" STREQUAL "")
    message (STATUS "using LIBDRM_SRC = ${LIBDRM_SRC}")
    include_directories (
        ${LIBDRM_SRC}
        ${LIBDRM_SRC}/include/drm
        ${LIBDRM_SRC}/intel
    )
endif ()

#################################################################################
# SOURCES
#################################################################################

# generic
set (LOCAL_SRC_FILES ${LOCAL_SRC_FILES}
    # mdapi metrics definitions
    ${BS_DIR_INSTRUMENTATION}/metrics_discovery/codegen/md_metrics_HSW.cpp
    ${BS_DIR_INSTRUMENTATION}/metrics_discovery/codegen/md_metrics_BDW.cpp
    ${BS_DIR_INSTRUMENTATION}/metrics_discovery/codegen/md_metrics_BXT.cpp
    ${BS_DIR_INSTRUMENTATION}/metrics_discovery/codegen/md_metrics_SKL_GT2.cpp
    ${BS_DIR_INSTRUMENTATION}/metrics_discovery/codegen/md_metrics_SKL_GT3.cpp
    ${BS_DIR_INSTRUMENTATION}/metrics_discovery/codegen/md_metrics_SKL_GT4.cpp
    ${BS_DIR_INSTRUMENTATION}/metrics_discovery/codegen/md_metrics_KBL_GT2.cpp
    ${BS_DIR_INSTRUMENTATION}/metrics_discovery/codegen/md_metrics_KBL_GT3.cpp
    ${BS_DIR_INSTRUMENTATION}/metrics_discovery/codegen/md_metrics_CFL_GT2.cpp
    ${BS_DIR_INSTRUMENTATION}/metrics_discovery/codegen/md_metrics_CFL_GT3.cpp
    ${BS_DIR_INSTRUMENTATION}/metrics_discovery/codegen/md_metrics_GLK.cpp
    ${BS_DIR_INSTRUMENTATION}/metrics_discovery/codegen/md_metrics_ICL.cpp
    ${BS_DIR_INSTRUMENTATION}/metrics_discovery/codegen/md_metrics_EHL.cpp
    ${BS_DIR_INSTRUMENTATION}/metrics_discovery/codegen/md_metrics_TGL.cpp
    ${BS_DIR_INSTRUMENTATION}/metrics_discovery/codegen/md_metrics_DG1.cpp
    ${BS_DIR_INSTRUMENTATION}/metrics_discovery/codegen/md_metrics.cpp
    # mdapi core
    ${BS_DIR_INSTRUMENTATION}/metrics_discovery/common/md_main.cpp
    ${BS_DIR_INSTRUMENTATION}/metrics_discovery/common/md_utils.cpp
    ${BS_DIR_INSTRUMENTATION}/metrics_discovery/common/md_internal.cpp
    ${BS_DIR_INSTRUMENTATION}/metrics_discovery/common/md_calculation.cpp
    ${BS_DIR_INSTRUMENTATION}/metrics_discovery/linux/md_driver_ifc_linux_perf.cpp
    # utils
    ${BS_DIR_INSTRUMENTATION}/utils/common/iu_debug.c
    ${BS_DIR_INSTRUMENTATION}/utils/linux/iu_std.cpp
    ${BS_DIR_INSTRUMENTATION}/utils/linux/iu_os.cpp
    # external
    ${BS_DIR_EXTERNAL}/mesa/src/intel/dev/gen_device_info.c
)

# mdapi internal files
if (("${BUILD_TYPE}" STREQUAL "release-internal") OR
    ("${BUILD_TYPE}" STREQUAL "debug"))
    set(MD_INTERNAL_BUILD TRUE)
endif ()

# mdapi internal metrics definitions
if (MD_INTERNAL_BUILD)
        set(LOCAL_SRC_FILES ${LOCAL_SRC_FILES}
#            currently none
        )
endif ()

#################################################################################
# FLAGS
#################################################################################

# select ENABLED_METRICS
# include only ENABLED_METRICS or all metrics if not defined.
if (DEFINED ENABLED_METRICS)
    message("Enable MDAPI metrics for platforms: ${ENABLED_METRICS}")
    foreach (hwPlatform ${ENABLED_METRICS})
        add_definitions(-DMD_INCLUDE_${hwPlatform}_METRICS)
    endforeach ()
else ()
    message("Enable MDAPI metrics for platforms: ALL")
endif ()

# configuration specific defines
if ("${BUILD_TYPE}" STREQUAL "release")
    add_definitions(-DNDEBUG)
    # security compiler flags
    add_definitions(-O2 -D_FORTIFY_SOURCE=2 -Wformat)
elseif ("${BUILD_TYPE}" STREQUAL "release-internal")
    add_definitions(-D_RELEASE_INTERNAL)
    add_definitions(-DNDEBUG)
elseif ("${BUILD_TYPE}" STREQUAL "debug")
    add_definitions(-D_DEBUG)
endif ()

# platform specific
if ("${PLATFORM}" STREQUAL "linux")
    # linux
    add_definitions(-fdiagnostics-color=auto) # color=auto only to colorize compiler output
    add_definitions(-fstack-protector)
    add_definitions(-fvisibility=hidden)      # to disable exporting internal symbols (by default gcc exports all symbols)
    add_definitions(-Wno-unused)
    add_definitions(-Wno-strict-aliasing)
    add_definitions(-Wno-unknown-pragmas)
    add_definitions(-Wno-comment)
    add_definitions(-msse)
    if ("${BUILD_TYPE}" STREQUAL "release" OR
        "${BUILD_TYPE}" STREQUAL "release-internal")
        add_definitions(-Wformat-security)
    elseif ("${BUILD_TYPE}" STREQUAL "debug")
        add_definitions(-g)
    endif ()
    # C++ specific
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-reorder")
    # support for cpp11
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
endif ()

# common
add_definitions(-DMD_USE_PERF)
add_definitions(-DMETRICS_DISCOVERY)
add_definitions(-fno-inline)
add_definitions(-Wall)

# to enable x86 build on x64 host
if ("${TARGET_ARCH}" STREQUAL "32")
    # flag is needed for both, compiler and linker
    add_definitions(-m32)
    set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -m32")
# to enable x64 build on x86 host
elseif("${TARGET_ARCH}" STREQUAL "64" AND ${HOST_ARCH} STREQUAL "32")
    # flag is needed for both, compiler and linker
    add_definitions(-m64)
    set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -m64")
endif ()

#################################################################################
# LINK LIBS
#################################################################################

set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -Wl,--no-undefined")
set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -Wl,-z,defs")
set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -Wl,-z,noexecstack")
set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -Wl,-z,relro")
set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -Wl,-z,now")

if ("${PLATFORM}" STREQUAL "linux")
    # linux
    add_library(${PROJECT_NAME} SHARED ${LOCAL_SRC_FILES})
    target_link_libraries(
        ${PROJECT_NAME}                 # metrics_discovery
        ${DRM_LIB_PATH}                 # drm
        rt
        pthread
        stdc++
    )
endif ()

set_property(TARGET ${PROJECT_NAME} PROPERTY POSITION_INDEPENDENT_CODE ON)
set_metrics_discovery_version()
set_property(TARGET ${PROJECT_NAME} PROPERTY VERSION ${MD_VERSION})
set_property(TARGET ${PROJECT_NAME} PROPERTY SOVERSION ${MD_VERSION_MAJOR})
set_property(TARGET ${PROJECT_NAME} PROPERTY OUTPUT_NAME ${LIB})

#################################################################################
# INSTALLER
#################################################################################

set (API_HEADER_LOCATION "${BS_DIR_INC}/common/instrumentation/api/metrics_discovery_api.h")

if (${PLATFORM} STREQUAL linux) # linux
    include(GNUInstallDirs)

    if (${TARGET_ARCH} STREQUAL 64)
        set (CPACK_PACKAGE_ARCHITECTURE        "x86_64")
        set (CPACK_DEBIAN_PACKAGE_ARCHITECTURE "amd64")
        set (CPACK_RPM_PACKAGE_ARCHITECTURE    "x86_64")
    else ()
        set (CPACK_PACKAGE_ARCHITECTURE        "i386")
        set (CPACK_DEBIAN_PACKAGE_ARCHITECTURE "i386")
        set (CPACK_RPM_PACKAGE_ARCHITECTURE    "i386")
    endif ()

    # pkg-config
    set (PC_FILE ${LIB_NAME}.pc)
    set (PC_IN ${PC_FILE}.in)

    configure_file (${PC_IN} ${CMAKE_BINARY_DIR}/${PC_FILE} @ONLY)

    install (FILES ${CMAKE_BINARY_DIR}/${PC_FILE}
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig
        COMPONENT metrics-discovery-pkgconfig
    )

    # header
    install (FILES ${API_HEADER_LOCATION}
        DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
        COMPONENT metrics-discovery-devel
    )

    # library
    install (TARGETS ${PROJECT_NAME}
        LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
        COMPONENT metrics-discovery
        NAMELINK_SKIP
    )

    install (TARGETS ${PROJECT_NAME}
        LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
        COMPONENT metrics-discovery-devel
        NAMELINK_ONLY
    )

    # software package generator
    if (LIB_CPACK_GENERATOR)
        set (CPACK_GENERATOR ${LIB_CPACK_GENERATOR})
    else ()
        if (EXISTS "/etc/debian_version")
            set (CPACK_GENERATOR "DEB")
        elseif (EXISTS "/etc/redhat-release")
            set (CPACK_GENERATOR "RPM")
        else ()
            set (CPACK_GENERATOR   "TXZ")
            set (CPACK_SET_DESTDIR ON)
        endif ()
    endif ()

    set (CPACK_DEBIAN_PACKAGE_MAINTAINER   "intel")
    set (CPACK_PACKAGE_NAME                "intel")
    set (CPACK_PACKAGE_VERSION             ${MD_VERSION})
    set (CPACK_PACKAGE_INSTALL_DIRECTORY   ${CMAKE_INSTALL_PREFIX})
    set (CPACK_DEBIAN_PACKAGE_DESCRIPTION  "Intel(R) Metrics Discovery Application Programming Interface")
    set (CPACK_DEBIAN_PACKAGE_SHLIBDEPS    ON)
    set (CPACK_RPM_PACKAGE_RELOCATABLE     TRUE)

    if (DEFINED LIB_PACKAGE_RELEASE)
        set (CPACK_DEBIAN_PACKAGE_RELEASE ${LIB_PACKAGE_RELEASE})
        set (CPACK_RPM_PACKAGE_RELEASE    ${LIB_PACKAGE_RELEASE})
    else ()
        set (CPACK_DEBIAN_PACKAGE_RELEASE 1)
        set (CPACK_RPM_PACKAGE_RELEASE    1)
    endif ()

    set (CPACK_DEBIAN_METRICS-DISCOVERY-PKGCONFIG_FILE_NAME  "intel-metrics-discovery-pkgconfig-${CPACK_PACKAGE_VERSION}-${CPACK_DEBIAN_PACKAGE_RELEASE}_${CPACK_DEBIAN_PACKAGE_ARCHITECTURE}.deb")
    set (CPACK_RPM_METRICS-DISCOVERY-PKGCONFIG_FILE_NAME     "intel-metrics-discovery-pkgconfig-${CPACK_PACKAGE_VERSION}-${CPACK_RPM_PACKAGE_RELEASE}.${CPACK_RPM_PACKAGE_ARCHITECTURE}.rpm")
    set (CPACK_ARCHIVE_METRICS-DISCOVERY-PKGCONFIG_FILE_NAME "intel-metrics-discovery-pkgconfig-${CPACK_PACKAGE_VERSION}-${CPACK_DEBIAN_PACKAGE_RELEASE}.${CPACK_PACKAGE_ARCHITECTURE}")

    set (CPACK_DEBIAN_METRICS-DISCOVERY-DEVEL_FILE_NAME  "intel-metrics-discovery-devel-${CPACK_PACKAGE_VERSION}-${CPACK_DEBIAN_PACKAGE_RELEASE}_${CPACK_DEBIAN_PACKAGE_ARCHITECTURE}.deb")
    set (CPACK_RPM_METRICS-DISCOVERY-DEVEL_FILE_NAME     "intel-metrics-discovery-devel-${CPACK_PACKAGE_VERSION}-${CPACK_RPM_PACKAGE_RELEASE}.${CPACK_RPM_PACKAGE_ARCHITECTURE}.rpm")
    set (CPACK_ARCHIVE_METRICS-DISCOVERY-DEVEL_FILE_NAME "intel-metrics-discovery-devel-${CPACK_PACKAGE_VERSION}-${CPACK_DEBIAN_PACKAGE_RELEASE}.${CPACK_PACKAGE_ARCHITECTURE}")

    set (CPACK_DEBIAN_METRICS-DISCOVERY_FILE_NAME  "intel-metrics-discovery-${CPACK_PACKAGE_VERSION}-${CPACK_DEBIAN_PACKAGE_RELEASE}_${CPACK_DEBIAN_PACKAGE_ARCHITECTURE}.deb")
    set (CPACK_RPM_METRICS-DISCOVERY_FILE_NAME     "intel-metrics-discovery-${CPACK_PACKAGE_VERSION}-${CPACK_RPM_PACKAGE_RELEASE}.${CPACK_RPM_PACKAGE_ARCHITECTURE}.rpm")
    set (CPACK_ARCHIVE_METRICS-DISCOVERY_FILE_NAME "intel-metrics-discovery-${CPACK_PACKAGE_VERSION}-${CPACK_DEBIAN_PACKAGE_RELEASE}.${CPACK_PACKAGE_ARCHITECTURE}")

    set(CPACK_DEBIAN_METRICS-DISCOVERY-DEVEL_PACKAGE_DEPENDS "intel-metrics-discovery(=${CPACK_PACKAGE_VERSION}-${CPACK_DEBIAN_PACKAGE_RELEASE})")
    set(CPACK_RPM_METRICS-DISCOVERY-DEVEL_PACKAGE_REQUIRES "intel-metrics-discovery = ${CPACK_PACKAGE_VERSION}-${CPACK_RPM_PACKAGE_RELEASE}")

    set (CPACK_DEB_COMPONENT_INSTALL     ON)
    set (CPACK_RPM_COMPONENT_INSTALL     ON)
    set (CPACK_ARCHIVE_COMPONENT_INSTALL ON)

    set (CPACK_COMPONENTS_ALL metrics-discovery-pkgconfig metrics-discovery-devel metrics-discovery)

    include (CPack)
endif ()

#################################################################################
# DEBUG
#################################################################################

message ("INFO: PROJECT_NAME           = ${PROJECT_NAME}")
message ("INFO: VERSION                = ${MD_VERSION}")
message ("INFO: BUILD_TYPE             = ${BUILD_TYPE}")
message ("INFO: PLATFORM               = ${PLATFORM}")
message ("INFO: HOST_ARCH              = ${HOST_ARCH}")
message ("INFO: TARGET_ARCH            = ${TARGET_ARCH}")
message ("INFO: LINUX_DISTRO           = ${LINUX_DISTRO}")
message ("INFO: DUMP_DIR               = ${DUMP_DIR}")
message ("INFO: LIB_INSTALL_DIR        = ${CMAKE_INSTALL_FULL_LIBDIR}")
message ("INFO: LIBDRM_SRC             = ${LIBDRM_SRC}")
message ("INFO: DRM_LIB_PATH           = ${DRM_LIB_PATH}")
message ("INFO: BS_DIR_INSTRUMENTATION = ${BS_DIR_INSTRUMENTATION}")
message ("INFO: BS_DIR_INC             = ${BS_DIR_INC}")
message ("INFO: BS_DIR_EXTERNAL        = ${BS_DIR_EXTERNAL}")
message ("INFO: CMAKE_C_FLAGS          = ${CMAKE_C_FLAGS}")
message ("INFO: CMAKE_CXX_FLAGS        = ${CMAKE_CXX_FLAGS}")
