cmake_minimum_required(VERSION 3.13)

project(cnats)
include(CTest)
include(FindPackageHandleStandardArgs)
include(CMakePackageConfigHelpers)

# Uncomment to have the build process verbose
# set(CMAKE_VERBOSE_MAKEFILE TRUE)

set(CMAKE_C_STANDARD 99)
set(CMAKE_C_STANDARD_REQUIRED ON)

# Set output directories for libraries and executables.
# This is important for Windows builds to have the DLLs in the same directory as the executables.
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)

# Set a default build type if none was specified
set(default_build_type "Release")

if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  message(STATUS "Setting build type to '${default_build_type}' as none was specified.")
  set(CMAKE_BUILD_TYPE "${default_build_type}" CACHE
      STRING "Choose the type of build." FORCE)
  # Set the possible values of build type for cmake-gui
  set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS
    "Release" "Debug" "MinSizeRel" "RelWithDebInfo")
endif()

option(NATS_UPDATE_VERSION "Update the version file" OFF)
option(NATS_UPDATE_DOC "Update the doc template file" OFF)
option(NATS_COVERAGE "Code coverage" OFF)
option(NATS_BUILD_NO_SPIN "Enable if spin code does not compile on your arch" OFF)
option(NATS_BUILD_WITH_TLS "Build with TLS support" ON)
option(NATS_BUILD_TLS_FORCE_HOST_VERIFY "Forces hostname verification" ON)
option(NATS_BUILD_TLS_USE_OPENSSL_1_1_API "Build for OpenSSL 1.1+" ON)
option(NATS_BUILD_USE_SODIUM "Build using libsodium library" OFF)
option(NATS_BUILD_EXAMPLES "Build examples" ON)
option(NATS_BUILD_LIBUV_EXAMPLE "Build libuv examples" OFF)
option(NATS_BUILD_LIBEVENT_EXAMPLE "Build libevent examples" OFF)
option(NATS_BUILD_STATIC_EXAMPLES "Statically link examples" OFF)

# Streaming is deprecated, but we still support it for now.
# We will remove it in the future. If you are starting a new project,
# please use JetStream. Another reason for disabling this is that
# building with streaming requires protobuf-c and building on Windows
# is not as straightforward as building without it.
option(NATS_BUILD_STREAMING "Build NATS Streaming" OFF)

option(NATS_BUILD_NO_PREFIX_CONNSTS "No prefix for connection status enum" OFF)
option(NATS_BUILD_LIB_STATIC "Build static library" ON)
option(NATS_BUILD_LIB_SHARED "Build shared library" ON)
option(NATS_COMPILER_HARDENING "Compiler hardening flags" OFF)
option(NATS_WITH_EXPERIMENTAL "Build with EXPERIMENTAL API support" OFF)
if(UNIX AND APPLE)
  option(CMAKE_MACOSX_RPATH "Build with macOS RPath" ON)
endif()

if(DEFINED ENV{NATS_SANITIZE})
  set(NATS_STANITIZE "$ENV{NATS_SANITIZE}")
  mark_as_advanced(NATS_SANITIZE)
endif()

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${CMAKE_BINARY_DIR})

if(NATS_COVERAGE)
  message("-- Coverage Enabled")
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} --coverage -fprofile-arcs -ftest-coverage")
endif(NATS_COVERAGE)

if(NATS_BUILD_WITH_TLS)
  set(OPENSSL_USE_STATIC_LIBS ${NATS_BUILD_OPENSSL_STATIC_LIBS})
  find_package(OpenSSL REQUIRED)
endif(NATS_BUILD_WITH_TLS)

set(LIBUV_DIR "" CACHE PATH "Libuv install directory")
set(LIBEVENT_DIR "" CACHE PATH "Libevent install directory")

set(NATS_DOC_PROJECT_NAME "NATS C Client with JetStream support")
if(NATS_BUILD_STREAMING)
  IF(DEFINED ENV{NATS_PROTOBUF_DIR})
    SET(NATS_PROTOBUF_DIR "$ENV{NATS_PROTOBUF_DIR}")
  ENDIF()
  FIND_PATH(
    NATS_PROTOBUF_INCLUDE_DIR
      protobuf-c/protobuf-c.h
    HINTS
      ${NATS_PROTOBUF_DIR}
  )
  FIND_LIBRARY(NATS_PROTOBUF_LIBRARY
    NAMES libprotobuf-c.so libprotobuf-c.dylib libprotobuf-c.a protobuf-c.lib protobuf-c.dll
    HINTS ${NATS_PROTOBUF_DIR}
  )
  FIND_PACKAGE_HANDLE_STANDARD_ARGS(libprotobuf-c DEFAULT_MSG
    NATS_PROTOBUF_INCLUDE_DIR
    NATS_PROTOBUF_LIBRARY
  )
  IF(libprotobuf-c_FOUND)
    SET(NATS_PROTOBUF_INCLUDE_DIRS ${NATS_PROTOBUF_INCLUDE_DIR})
    SET(NATS_PROTOBUF_LIBRARIES ${NATS_PROTOBUF_LIBRARY})
    MARK_AS_ADVANCED(
      NATS_PROTOBUF_LIBRARY
      NATS_PROTOBUF_INCLUDE_DIR
      NATS_PROTOBUF_DIR
    )
  ELSE()
    SET(NATS_PROTOBUF_DIR "" CACHE STRING
      "An optional hint to a directory for finding `libprotobuf-c`"
    )
    MESSAGE(FATAL_ERROR
      "Could not find libprotobuf-c package. Check build instructions: https://github.com/nats-io/nats.c#building-with-streaming")
  ENDIF()
  add_definitions(-DNATS_HAS_STREAMING)
  set(NATS_DOC_INCLUDE_STREAMING "NATS_HAS_STREAMING")
  set(NATS_DOC_PROJECT_NAME "NATS C Client with JetStream and Streaming support")
endif(NATS_BUILD_STREAMING)

if(NATS_BUILD_USE_SODIUM)
  IF(DEFINED ENV{NATS_SODIUM_DIR})
    SET(NATS_SODIUM_DIR "$ENV{NATS_SODIUM_DIR}")
  ENDIF()
  FIND_PATH(
    NATS_SODIUM_INCLUDE_DIR
      sodium.h
    HINTS
      ${NATS_SODIUM_DIR}
  )
  FIND_LIBRARY(NATS_SODIUM_LIBRARY
    NAMES libsodium.so libsodium.dylib libsodium.a libsodium.lib libsodium.dll
    HINTS ${NATS_SODIUM_DIR}
  )
  FIND_PACKAGE_HANDLE_STANDARD_ARGS(libsodium DEFAULT_MSG
    NATS_SODIUM_INCLUDE_DIR
    NATS_SODIUM_LIBRARY
  )
  IF(libsodium_FOUND)
    SET(NATS_SODIUM_INCLUDE_DIRS ${NATS_SODIUM_INCLUDE_DIR})
    SET(NATS_SODIUM_LIBRARIES ${NATS_SODIUM_LIBRARY})
    MARK_AS_ADVANCED(
      NATS_SODIUM_LIBRARY
      NATS_SODIUM_INCLUDE_DIR
      NATS_SODIUM_DIR
    )
  ELSE()
    SET(NATS_SODIUM_DIR "" CACHE STRING
      "An optional hint to a directory for finding `libsodium`"
    )
    MESSAGE(FATAL_ERROR
      "Could not find libsodium package. Check build instructions: https://github.com/nats-io/nats.c#building-with-libsodium")
  ENDIF()
  add_definitions(-DNATS_USE_LIBSODIUM)
endif(NATS_BUILD_USE_SODIUM)

if(NATS_BUILD_NO_PREFIX_CONNSTS)
  add_definitions(-DNATS_CONN_STATUS_NO_PREFIX)
endif(NATS_BUILD_NO_PREFIX_CONNSTS)

if(NATS_WITH_EXPERIMENTAL)
  add_definitions(-DNATS_WITH_EXPERIMENTAL)
endif(NATS_WITH_EXPERIMENTAL)

# Platform specific settings
if(UNIX)
  #---------------------------------------------------------------------------
  # Define NATS cache variables that override the CMAKE and MEMCHECK variables
  #---------------------------------------------------------------------------
  include(GNUInstallDirs)

  set(NATS_LIBDIR ${CMAKE_INSTALL_LIBDIR})
  set(NATS_INCLUDE_DIR ${CMAKE_INSTALL_INCLUDEDIR})

  set(NATS_BUILD_ARCH "64" CACHE STRING "32 for 32bits builds")

  find_program(MEMORYCHECK_COMMAND valgrind)
  set(NATS_MEMCHECK_CMD_OPTS "--leak-check=full --track-fds=yes --show-reachable=yes --num-callers=50" CACHE STRING "Memcheck options")
  set(MEMORYCHECK_COMMAND_OPTIONS ${NATS_MEMCHECK_CMD_OPTS} CACHE INTERNAL "")

  set(NATS_COMMON_C_FLAGS "-std=c99 -pedantic")

  # Some compiler hardening flags.
  if(NATS_COMPILER_HARDENING)
     set(NATS_COMMON_C_FLAGS "${NATS_COMMON_C_FLAGS} -fPIE -fstack-protector-all -D_FORTIFY_SOURCE=2 -O1")
     # Works only with GCC.
     if("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU")
       set(NATS_COMMON_C_FLAGS "${NATS_COMMON_C_FLAGS} -Wl,-z,relro,-z,now -pie")
     endif("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU")
  endif(NATS_COMPILER_HARDENING)

  set(NATS_WARNINGS "-fstrict-aliasing -Wall -W -Wno-unused-parameter -Wno-unused-function -Wstrict-prototypes -Wwrite-strings")
  set(NATS_PLATFORM_INCLUDE "unix")

  if(APPLE)
    set(NATS_OS "DARWIN")
  else(APPLE)
    set(NATS_OS "LINUX")
    set(NATS_USE_PTHREAD "-pthread")
    if(NOT ANDROID)
      set(NATS_EXTRA_LIB "rt -lpthread")
    endif()
  endif(APPLE)
  if (${NATS_BUILD_ARCH} MATCHES "32")
    if(NOT APPLE)
      message("-----------------------------------------------------------")
      message("If build fails, you probably need to install libc6-dev-i386")
      message("apt-get install libc6-dev-i386")
      message("-----------------------------------------------------------")
    endif(NOT APPLE)

    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -m32")
    set(CMAKE_C_LINKER_FLAGS "${CMAKE_C_LINKER_FLAGS} -m32")
  endif(${NATS_BUILD_ARCH} MATCHES "32")

elseif(WIN32)
  set(NATS_LIBDIR "lib")
  set(NATS_INCLUDE_DIR "include")
  set(NATS_EXTRA_LIB "ws2_32")
  set(NATS_OS "_WIN32")
  set(NATS_PLATFORM_INCLUDE "win")

  # Warning control.
  if(MSVC)
    # Set warning level to maximum, then disable:
    # /wd4100 unreferenced formal parameter
    # /wd4200 nonstandard extension used: zero-sized array in struct/union
    # /wd4130 logical operation on address of string constant
    # /wd4127 conditional expression is constant
    add_compile_options(/W4 /wd4100 /wd4200 /wd4130 /wd4127)
  else()
    add_compile_options(-Wall -Wextra -Wno-unused-parameter -Wno-pedantic -Wno-address -Wno-constant-conditional)
  endif()

  if(sodium_USE_STATIC_LIBS)
    add_definitions(
      -DSODIUM_STATIC
      -DSODIUM_EXPORT=
    )
  endif(sodium_USE_STATIC_LIBS)
endif(UNIX)

set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${NATS_COMMON_C_FLAGS} ${NATS_USE_PTHREAD} ${NATS_WARNINGS}")
if(NATS_SANITIZE)
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fno-omit-frame-pointer -fno-optimize-sibling-calls")
endif(NATS_SANITIZE)

if(UNIX)
  if(APPLE)
  else(APPLE)
    add_definitions(-D_GNU_SOURCE)
    if(NATS_BUILD_NO_SPIN)
      add_definitions(-DNATS_NO_SPIN)
    endif(NATS_BUILD_NO_SPIN)
  endif(APPLE)
endif(UNIX)
add_definitions(-D${NATS_OS})
add_definitions(-D_REENTRANT)
if(NATS_BUILD_WITH_TLS)
  add_definitions(-DNATS_HAS_TLS)
  if(NATS_BUILD_TLS_USE_OPENSSL_1_1_API)
    add_definitions(-DNATS_USE_OPENSSL_1_1)
  endif(NATS_BUILD_TLS_USE_OPENSSL_1_1_API)
  if(NATS_BUILD_TLS_FORCE_HOST_VERIFY)
    add_definitions(-DNATS_FORCE_HOST_VERIFICATION)
  endif(NATS_BUILD_TLS_FORCE_HOST_VERIFY)
  set(NATS_DOC_INCLUDE_TLS "NATS_HAS_TLS")
endif(NATS_BUILD_WITH_TLS)


#------------
# Versionning and Doc

set(NATS_VERSION_MAJOR  3)
set(NATS_VERSION_MINOR  11)
set(NATS_VERSION_PATCH  0)
set(NATS_VERSION_SUFFIX "-beta")

set(NATS_VERSION_REQUIRED_NUMBER 0x030A00)

write_basic_package_version_file(
  "${PROJECT_BINARY_DIR}/cnats-config-version.cmake"
  COMPATIBILITY SameMinorVersion
  VERSION ${NATS_VERSION_MAJOR}.${NATS_VERSION_MINOR}.${NATS_VERSION_PATCH}${NATS_VERSION_SUFFIX})

if(NATS_UPDATE_VERSION OR NATS_UPDATE_DOC)
configure_file(
	${CMAKE_CURRENT_SOURCE_DIR}/src/version.h.in
	${CMAKE_CURRENT_SOURCE_DIR}/src/version.h
	@ONLY)

configure_file(
	${CMAKE_SOURCE_DIR}/doc/DoxyFile.NATS.Client.in
	${CMAKE_SOURCE_DIR}/doc/DoxyFile.NATS.Client
	@ONLY)
endif(NATS_UPDATE_VERSION OR NATS_UPDATE_DOC)
#------------

#------------
# pkg-config
if(UNIX)
  configure_file(
    ${PROJECT_SOURCE_DIR}/src/libnats.pc.in
    ${PROJECT_BINARY_DIR}/libnats.pc
    @ONLY
  )
  install (
    FILES "${PROJECT_BINARY_DIR}/libnats.pc"
    DESTINATION "${CMAKE_INSTALL_LIBDIR}/pkgconfig")
endif(UNIX)
#------------

#----------------------------
# Add the project directories

add_subdirectory(src)
add_subdirectory(examples)
add_subdirectory(examples/getstarted)
if(NATS_BUILD_STREAMING)
  add_subdirectory(examples/stan)
endif()
add_subdirectory(test)
add_subdirectory(test/dylib)
add_subdirectory(test/check_cpp)
#----------------------------
