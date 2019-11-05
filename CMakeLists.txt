cmake_minimum_required(VERSION 3.0)
project(Hiawatha VERSION 10.9 LANGUAGES C)

# Options
option(ENABLE_TLS      "Enable TLS (mbed TLS) support in Hiawatha." on)
option(USE_SYSTEM_MBEDTLS "Use the system's mbed TLS (>=2.0.0) library." off)

# CMake common modules
include(CheckIncludeFile)
include(CheckIncludeFiles)
include(CheckFunctionExists)
include(CheckLibraryExists)
include(CheckSymbolExists)

if(ENABLE_TLS)
    # Check for mbedTLS
    if(USE_SYSTEM_MBEDTLS)
        message(WARNING "Make sure the mbed TLS shared library has been built with the MBEDTLS_THREADING_PTHREAD and MBEDTLS_THREADING_C flags set. Otherwise, it may crash the Hiawatha webserver.")
        find_library(MBEDCRYPTO "mbedcrypto")
        find_library(MBEX509 "mbedx509")
        find_library(MBEDTLS "mbedtls")
        set(SYSTEM_MBEDTLS_LIBRARIES ${MBEDCRYPTO} ${MBEDX509} ${MBEDTLS})
    else()
        option(USE_STATIC_MBEDTLS_LIBRARY "Build mbed TLS static library." off)
        option(USE_SHARED_MBEDTLS_LIBRARY "Build mbed TLS shared library." on)
        if (CMAKE_SYSTEM MATCHES "OpenBSD")
            option(LINK_WITH_PTHREAD "Explicitly link mbed TLS library to pthread." on)
        endif()
        cmake_policy(SET CMP0048 NEW)
        add_subdirectory(mbedtls)
        include_directories(mbedtls/include)
        set(MBEDTLS_LIBRARIES "mbedtls" "mbedx509" "mbedcrypto")
    endif()
endif(ENABLE_TLS)

# POSIX thread API (required)
find_package(Threads REQUIRED)
set(THREADS_PREFER_PTHREAD_FLAG ON)

# For auto-generated *.h files
include_directories(${CMAKE_CURRENT_BINARY_DIR})

# Binaries
aux_source_directory("${CMAKE_CURRENT_LIST_DIR}/src" main_src)
add_executable(main ${main_src})
target_link_libraries(main Threads::Threads)
if(ENABLE_TLS)
    if(USE_SYSTEM_MBEDTLS)
        target_link_libraries(main ${SYSTEM_MBEDTLS_LIBRARIES})
    else()
        target_link_libraries(main ${MBEDTLS_LIBRARIES})
    endif()
endif(ENABLE_TLS)

# Where to install libs and exe programs
include(GNUInstallDirs)
if(ENABLE_TLS)
    set(CMAKE_INSTALL_NAME_DIR ${CMAKE_INSTALL_FULL_LIBDIR}/hiawatha)
    set(LIB_INSTALL_DIR ${CMAKE_INSTALL_LIBDIR}/hiawatha)
    if (APPLE)
        set(CMAKE_MACOSX_RPATH ${CMAKE_MACOSX_RPATH};${CMAKE_INSTALL_FULL_LIBDIR}/hiawatha)
    endif()
    if(NOT USE_SYSTEM_MBEDTLS)
        set_target_properties(main PROPERTIES INSTALL_RPATH ${CMAKE_INSTALL_FULL_LIBDIR}/hiawatha)
    endif()
endif(ENABLE_TLS)

# Installation
install(TARGETS main DESTINATION ${CMAKE_INSTALL_BINDIR})
