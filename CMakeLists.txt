cmake_minimum_required(VERSION 3.27 FATAL_ERROR)

project(idaxex)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_C_STANDARD 11)

include(CheckCCompilerFlag)
include(CheckCXXCompilerFlag)

function(idaxex_enable_aesni target_name)
  if(CMAKE_SYSTEM_PROCESSOR MATCHES "^(x86_64|amd64|AMD64|i[3-6]86|x86)$")
    check_c_compiler_flag("-maes" IDAXEX_C_HAS_MAES)
    check_cxx_compiler_flag("-maes" IDAXEX_CXX_HAS_MAES)
    if(IDAXEX_C_HAS_MAES AND IDAXEX_CXX_HAS_MAES)
      target_compile_options(${target_name} PRIVATE
        $<$<COMPILE_LANGUAGE:C>:-maes>
        $<$<COMPILE_LANGUAGE:CXX>:-maes>
      )
    endif()
  endif()
endfunction()

option(IDAXEX_BUILD_PLUGIN "Build the IDA loader plugin target" ON)
option(IDAXEX_BUILD_XEX1TOOL "Build the standalone xex1tool target" ON)

if(IDAXEX_BUILD_XEX1TOOL)
  add_subdirectory(xex1tool)
endif()

set(IDAXEX_IDASDK "$ENV{IDASDK}")
if(NOT IDAXEX_IDASDK AND EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/ida-sdk-9.3/src/include/ida.hpp")
  set(IDAXEX_IDASDK "${CMAKE_CURRENT_SOURCE_DIR}/ida-sdk-9.3/src")
  set(ENV{IDASDK} "${IDAXEX_IDASDK}")
endif()

set(IDAXEX_IDA_CMAKE "${IDAXEX_IDASDK}/ida-cmake/common.cmake")
set(IDAXEX_IDA_INCLUDE_DIR "${IDAXEX_IDASDK}/include")

if(NOT IDAXEX_BUILD_PLUGIN)
  return()
endif()

set(LOADER_NAME idaxex)
set(LOADER_SOURCES
  idaloader.cpp
  idaloader_xbe.cpp
  namegen.cpp
  namegen_xtlid.cpp
  formats/xbe.cpp
  formats/xex.cpp
  3rdparty/excrypt/src/excrypt_aes.c
  3rdparty/excrypt/src/rijndael.c
  3rdparty/excrypt/src/excrypt_sha.c
  3rdparty/lzx.cpp
  3rdparty/mspack/lzxd.c
  3rdparty/mspack/system.c
  3rdparty/XbSymbolDatabase/src/lib/libXbSymbolDatabase.c
  3rdparty/XbSymbolDatabase/src/OOVPADatabase/D3D8_OOVPA.c
  3rdparty/XbSymbolDatabase/src/OOVPADatabase/D3D8LTCG_OOVPA.c
  3rdparty/XbSymbolDatabase/src/OOVPADatabase/DSound_OOVPA.c
  3rdparty/XbSymbolDatabase/src/OOVPADatabase/JVS_OOVPA.c
  3rdparty/XbSymbolDatabase/src/OOVPADatabase/XActEng_OOVPA.c
  3rdparty/XbSymbolDatabase/src/OOVPADatabase/Xapi_OOVPA.c
  3rdparty/XbSymbolDatabase/src/OOVPADatabase/XGraphic_OOVPA.c
  3rdparty/XbSymbolDatabase/src/OOVPADatabase/XNet_OOVPA.c
  3rdparty/XbSymbolDatabase/src/OOVPADatabase/XOnline_OOVPA.c
)
set(LOADER_INCLUDE_DIRECTORIES
  3rdparty/excrypt/src
  3rdparty/XbSymbolDatabase/include
  3rdparty/XbSymbolDatabase/src/OOVPADatabase
)

if(IDAXEX_IDASDK AND EXISTS "${IDAXEX_IDA_CMAKE}")
  include("${IDAXEX_IDA_CMAKE}")
  list(APPEND LOADER_INCLUDE_DIRECTORIES "${IDAXEX_IDASDK}/ldr/pe")
  add_compile_definitions(IDALDR=1)
  generate()
  disable_ida_warnings(idaxex)
  idaxex_enable_aesni(idaxex)
  return()
endif()

if(NOT IDAXEX_IDASDK OR NOT EXISTS "${IDAXEX_IDA_INCLUDE_DIR}/loader.hpp")
  message(STATUS "Skipping idaxex plugin build: set IDASDK to an IDA SDK checkout (or place ida-sdk-9.3/src in the repo) to enable it.")
  return()
endif()

if(CMAKE_SYSTEM_NAME STREQUAL "Darwin")
  if(CMAKE_SYSTEM_PROCESSOR MATCHES "^(arm64|aarch64)$")
    set(IDAXEX_IDA_LIBDIR "${IDAXEX_IDASDK}/lib/arm64_mac_clang_64")
  elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "^(x86_64|amd64|AMD64)$")
    set(IDAXEX_IDA_LIBDIR "${IDAXEX_IDASDK}/lib/x64_mac_clang_64")
  endif()
elseif(CMAKE_SYSTEM_NAME STREQUAL "Linux")
  if(CMAKE_SYSTEM_PROCESSOR MATCHES "^(x86_64|amd64|AMD64)$")
    set(IDAXEX_IDA_LIBDIR "${IDAXEX_IDASDK}/lib/x64_linux_gcc_64")
  endif()
endif()

set(IDAXEX_IDA_LIBRARY "${IDAXEX_IDA_LIBDIR}/libida.dylib")
if(CMAKE_SYSTEM_NAME STREQUAL "Linux")
  set(IDAXEX_IDA_LIBRARY "${IDAXEX_IDA_LIBDIR}/libida.so")
endif()

if(NOT EXISTS "${IDAXEX_IDA_LIBRARY}")
  message(FATAL_ERROR "Could not locate the IDA kernel stub library for this platform at ${IDAXEX_IDA_LIBRARY}")
endif()

add_library(idaxex SHARED ${LOADER_SOURCES})
target_include_directories(idaxex PRIVATE
  ${LOADER_INCLUDE_DIRECTORIES}
  ${IDAXEX_IDA_INCLUDE_DIR}
  "${IDAXEX_IDASDK}/ldr/pe"
)
target_compile_definitions(idaxex PRIVATE
  IDALDR=1
  LOADER_COMPILE
  __EA64__
  NDEBUG
)

if(CMAKE_SYSTEM_NAME STREQUAL "Darwin")
  target_compile_definitions(idaxex PRIVATE __MAC__)
  if(CMAKE_SYSTEM_PROCESSOR MATCHES "^(arm64|aarch64)$")
    target_compile_definitions(idaxex PRIVATE __ARM__ __APPLE_SILICON__)
  elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "^(x86_64|amd64|AMD64)$")
    target_compile_definitions(idaxex PRIVATE __X64__)
  elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "^(i[3-6]86|x86)$")
    target_compile_definitions(idaxex PRIVATE __X86__)
  endif()
elseif(CMAKE_SYSTEM_NAME STREQUAL "Linux")
  target_compile_definitions(idaxex PRIVATE __LINUX__)
  if(CMAKE_SYSTEM_PROCESSOR MATCHES "^(x86_64|amd64|AMD64)$")
    target_compile_definitions(idaxex PRIVATE __X64__)
  elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "^(i[3-6]86|x86)$")
    target_compile_definitions(idaxex PRIVATE __X86__)
  endif()
endif()

target_link_libraries(idaxex PRIVATE "${IDAXEX_IDA_LIBRARY}")
set_target_properties(idaxex PROPERTIES
  PREFIX ""
  OUTPUT_NAME "idaxex"
  LIBRARY_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/loaders"
)

if(CMAKE_SYSTEM_NAME STREQUAL "Darwin")
  set_target_properties(idaxex PROPERTIES SUFFIX ".dylib")
  target_link_options(idaxex PRIVATE
    "-Wl,-install_name,idaxex.dylib"
    "-Wl,-rpath,@loader_path/.."
    "-Wl,-dead_strip"
  )
endif()

idaxex_enable_aesni(idaxex)
