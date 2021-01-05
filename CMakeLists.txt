cmake_minimum_required(VERSION 3.0)
enable_language(C)
project(libftl)

set(FTL_VERSION "0.10.1")

option(DISABLE_AUTO_INGEST "Set to TRUE to disable auto ingest feature which removes curl and jansson dependancies" FALSE)
message(STATUS "FTL DISABLE_AUTO_INGEST: " ${DISABLE_AUTO_INGEST})

option(FTL_STATIC_COMPILE "Set to TRUE if you want ftl to be compiled as a static lib. If TRUE, the program will want to statically link to the ftl cmake object." FALSE)
message(STATUS "FTL FTL_STATIC_COMPILE: " ${FTL_STATIC_COMPILE})

include(GNUInstallDirs)
find_package(Threads REQUIRED)
find_package(PkgConfig)

set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR})
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR})

if (DISABLE_AUTO_INGEST)
    add_definitions(-DDISABLE_AUTO_INGEST)
endif ()

# We will only try to include curl if we have auto ingest enabled.
if (NOT DISABLE_AUTO_INGEST)
    find_package(CURL)
endif ()
if (NOT CURL_FOUND AND NOT DISABLE_AUTO_INGEST)
    set(CURL_DISABLE_NTLM ON CACHE BOOL "Disabling NTLM")
    set(CURL_DISABLE_TELNET ON CACHE BOOL "Disabling Telnet")
    set(CURL_DISABLE_LDAP ON CACHE BOOL "Disabling Ldap")
    set(CURL_DISABLE_LDAPS ON CACHE BOOL "Disabling secure ldap")
    set(BUILD_CURL_EXE OFF CACHE BOOL "Building libcurl")
    set(HTTP_ONLY ON CACHE BOOL "using compiling HTTP")
    set(BUILD_TESTING OFF CACHE BOOL "Not building Tests")
    add_subdirectory(libcurl)
    set(CURL_INCLUDE_DIRS libcurl/include ${CMAKE_CURRENT_BINARY_DIR}/libcurl/include/curl)
    set(CURL_LIBRARIES libcurl)
    include_directories(${CURL_INCLUDE_DIRS})
endif ()

# We will only try to include lib jansson if auto ingest is enabled.
if (PKG_CONFIG_FOUND AND NOT DISABLE_AUTO_INGEST)
    pkg_check_modules(JANSSON jansson)
endif ()
if (NOT JANSSON_FOUND AND NOT DISABLE_AUTO_INGEST)
    set(JANSSON_BUILD_DOCS OFF CACHE BOOL "Jansson docs off")
    set(JANSSON_WITHOUT_TESTS ON CACHE BOOL "Jansson build without tests")
    set(JANSSON_EXAMPLES OFF CACHE BOOL "Jansson disable examples")
    set(USE_WINDOWS_CRYPTOAPI off)
    add_subdirectory(libjansson)
    set(JANSSON_INCLUDE_DIRS ${CMAKE_CURRENT_BINARY_DIR}/libjansson/include)
    set(JANSSON_LIBRARIES jansson)
    include_directories(${JANSSON_INCLUDE_DIRS})
endif ()

if (WIN32)
    #set(FTL_PLATFORM_LIBS kernel32 user32 gdi32 advapi32 )
    #set(FTL_PLATFORM_LIBS ws2_32 )
    set(FTLSDK_PLATFORM_FILES libftl/win32/socket.c
            libftl/win32/socket.h
            libftl/win32/threads.c
            libftl/win32/threads.h)
    include_directories(libftl/win32)
else ()
    set(FTLSDK_PLATFORM_FILES libftl/posix/socket.c
            libftl/posix/socket.h
            libftl/posix/threads.c
            libftl/posix/threads.h)
    include_directories(libftl/posix)
endif ()

# Figure out what kind of lib we should be producing.
if (FTL_STATIC_COMPILE)
    set(FTL_LIB_TYPE STATIC)
    add_definitions(-DFTL_STATIC_COMPILE=1)
else ()
    set(FTL_LIB_TYPE SHARED)
endif ()

add_library(ftl ${FTL_LIB_TYPE} libftl/hmac/hmac.c
        libftl/hmac/hmac.h
        libftl/hmac/sha2.c
        libftl/hmac/sha2.h
        libftl/gettimeofday/gettimeofday.c
        libftl/gettimeofday/gettimeofday.h
        libftl/ftl-sdk.c
        libftl/handshake.c
        libftl/ingest.c
        libftl/ftl_helpers.c
        libftl/media.c
        libftl/logging.c
        libftl/ftl.h
        libftl/ftl_private.h
        ${FTLSDK_PLATFORM_FILES})
include_directories(libftl libftl/gettimeofday)

set_target_properties(ftl PROPERTIES VERSION ${FTL_VERSION})
set_target_properties(ftl PROPERTIES SOVERSION 0)

target_link_libraries(ftl ${CURL_LIBRARIES} ${JANSSON_LIBRARIES})

if (WIN32)
    target_link_libraries(ftl ws2_32)
endif ()

configure_file(${CMAKE_CURRENT_SOURCE_DIR}/libftl.pc.in
        ${CMAKE_CURRENT_BINARY_DIR}/libftl.pc @ONLY)

# Install rules
install(TARGETS ftl DESTINATION ${CMAKE_INSTALL_LIBDIR})
install(FILES libftl/ftl.h DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/libftl)
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/libftl.pc
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig)
