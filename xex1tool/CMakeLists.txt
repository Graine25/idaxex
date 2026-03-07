cmake_minimum_required(VERSION 3.10)

project(xex1tool)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED True)

include(CheckCCompilerFlag)
include(CheckCXXCompilerFlag)

add_executable(xex1tool
  ../xex1tool.cpp
  ../namegen.cpp
  ../formats/xdbf.cpp
  ../formats/xex.cpp
  ../3rdparty/excrypt/src/excrypt_aes.c
  ../3rdparty/excrypt/src/rijndael.c
  ../3rdparty/excrypt/src/excrypt_bn.c
  ../3rdparty/excrypt/src/excrypt_bn_mod.cpp
  ../3rdparty/excrypt/src/excrypt_bn_sig.c
  ../3rdparty/excrypt/src/excrypt_mem.cpp
  ../3rdparty/excrypt/src/excrypt_rc4.c
  ../3rdparty/excrypt/src/excrypt_rotsum.c
  ../3rdparty/excrypt/src/excrypt_sha.c
  ../3rdparty/lzx.cpp
  ../3rdparty/mspack/lzxd.c
  ../3rdparty/mspack/system.c
)

target_include_directories(xex1tool PRIVATE ${PROJECT_SOURCE_DIR}/../3rdparty/excrypt/src)

if(CMAKE_SYSTEM_PROCESSOR MATCHES "^(x86_64|amd64|AMD64|i[3-6]86|x86)$")
  check_c_compiler_flag("-maes" XEX1TOOL_C_HAS_MAES)
  check_cxx_compiler_flag("-maes" XEX1TOOL_CXX_HAS_MAES)
  if(XEX1TOOL_C_HAS_MAES AND XEX1TOOL_CXX_HAS_MAES)
    target_compile_options(xex1tool PRIVATE
      $<$<COMPILE_LANGUAGE:C>:-maes>
      $<$<COMPILE_LANGUAGE:CXX>:-maes>
    )
  endif()
endif()
