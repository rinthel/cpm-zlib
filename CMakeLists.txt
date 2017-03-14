cmake_minimum_required(VERSION 2.8 FATAL_ERROR)

#-----------------------------------------------------------------------
# CPM configuration
#-----------------------------------------------------------------------
set(CPM_MODULE_NAME zlib)
set(CPM_LIB_TARGET_NAME ${CPM_MODULE_NAME})

if ((DEFINED CPM_DIR) AND (DEFINED CPM_UNIQUE_ID) AND (DEFINED CPM_TARGET_NAME))
  set(CPM_LIB_TARGET_NAME ${CPM_TARGET_NAME})
  set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${CPM_DIR})
  include(CPM)
else()
  set(CPM_DIR "${CMAKE_CURRENT_BINARY_DIR}/cpm-packages" CACHE TYPE STRING)
  find_package(Git)
  if(NOT GIT_FOUND)
    message(FATAL_ERROR "CPM requires Git.")
  endif()
  if (NOT EXISTS ${CPM_DIR}/CPM.cmake)
    execute_process(
      COMMAND "${GIT_EXECUTABLE}" clone https://github.com/iauns/cpm ${CPM_DIR}
      RESULT_VARIABLE error_code
      OUTPUT_VARIABLE head_sha)
    if(error_code)
      message(FATAL_ERROR "CPM failed to get the hash for HEAD")
    endif()
  endif()
  include(${CPM_DIR}/CPM.cmake)
endif()

# All externals *must* define this.
CPM_ForceOnlyOneModuleVersion()

CPM_InitModule(${CPM_MODULE_NAME})

#------------------------------------------------------------------------------
# Native File Dialog
#------------------------------------------------------------------------------

set(ZLIB_DIR ${CMAKE_CURRENT_SOURCE_DIR}/3rdParty/zlib)
set(ZLIB_BIN_DIR ${CMAKE_CURRENT_BINARY_DIR}/3rdParty/zlib)

CPM_EnsureRepoIsCurrent(
  TARGET_DIR ${ZLIB_DIR}
  GIT_REPOSITORY "https://github.com/madler/zlib.git"
  USE_CACHING TRUE
  )

project(zlib C)

set(VERSION "1.2.11")

if(MSVC)
    set(CMAKE_DEBUG_POSTFIX "d")
    add_definitions(-D_CRT_SECURE_NO_DEPRECATE)
    add_definitions(-D_CRT_NONSTDC_NO_DEPRECATE)
    include_directories(${ZLIB_DIR})
endif()

set(ZLIB_PC ${CMAKE_CURRENT_BINARY_DIR}/zlib.pc)
configure_file( ${ZLIB_DIR}/zlib.pc.cmakein
    ${ZLIB_PC} @ONLY)
configure_file( ${ZLIB_DIR}/zconf.h.cmakein
    ${CMAKE_CURRENT_BINARY_DIR}/zconf.h @ONLY)
include_directories(${CMAKE_CURRENT_BINARY_DIR} ${CMAKE_SOURCE_DIR})

#============================================================================
# zlib
#============================================================================

set(ZLIB_PUBLIC_HDRS
    ${ZLIB_DIR}/zconf.h
    ${ZLIB_DIR}/zlib.h
)
set(ZLIB_PRIVATE_HDRS
    ${ZLIB_DIR}/crc32.h
    ${ZLIB_DIR}/deflate.h
    ${ZLIB_DIR}/gzguts.h
    ${ZLIB_DIR}/inffast.h
    ${ZLIB_DIR}/inffixed.h
    ${ZLIB_DIR}/inflate.h
    ${ZLIB_DIR}/inftrees.h
    ${ZLIB_DIR}/trees.h
    ${ZLIB_DIR}/zutil.h
)
set(ZLIB_SRCS
    ${ZLIB_DIR}/adler32.c
    ${ZLIB_DIR}/compress.c
    ${ZLIB_DIR}/crc32.c
    ${ZLIB_DIR}/deflate.c
    ${ZLIB_DIR}/gzclose.c
    ${ZLIB_DIR}/gzlib.c
    ${ZLIB_DIR}/gzread.c
    ${ZLIB_DIR}/gzwrite.c
    ${ZLIB_DIR}/inflate.c
    ${ZLIB_DIR}/infback.c
    ${ZLIB_DIR}/inftrees.c
    ${ZLIB_DIR}/inffast.c
    ${ZLIB_DIR}/trees.c
    ${ZLIB_DIR}/uncompr.c
    ${ZLIB_DIR}/zutil.c
)

if(NOT MINGW)
    set(ZLIB_DLL_SRCS
        win32/zlib1.rc # If present will override custom build rule below.
    )
endif()

if(CMAKE_COMPILER_IS_GNUCC)
    if(ASM686)
        set(ZLIB_ASMS contrib/asm686/match.S)
    elseif (AMD64)
        set(ZLIB_ASMS contrib/amd64/amd64-match.S)
    endif ()

  if(ZLIB_ASMS)
    add_definitions(-DASMV)
    set_source_files_properties(${ZLIB_ASMS} PROPERTIES LANGUAGE C COMPILE_FLAGS -DNO_UNDERLINE)
  endif()
endif()

if(MSVC)
    if(ASM686)
    ENABLE_LANGUAGE(ASM_MASM)
        set(ZLIB_ASMS
      contrib/masmx86/inffas32.asm
      contrib/masmx86/match686.asm
    )
    elseif (AMD64)
    ENABLE_LANGUAGE(ASM_MASM)
        set(ZLIB_ASMS
      contrib/masmx64/gvmat64.asm
      contrib/masmx64/inffasx64.asm
    )
    endif()

  if(ZLIB_ASMS)
    add_definitions(-DASMV -DASMINF)
  endif()
endif()

# parse the full version number from zlib.h and include in ZLIB_FULL_VERSION
file(READ ${ZLIB_DIR}/zlib.h _zlib_h_contents)
string(REGEX REPLACE ".*#define[ \t]+ZLIB_VERSION[ \t]+\"([-0-9A-Za-z.]+)\".*"
    "\\1" ZLIB_FULL_VERSION ${_zlib_h_contents})

if(MINGW)
    # This gets us DLL resource information when compiling on MinGW.
    if(NOT CMAKE_RC_COMPILER)
        set(CMAKE_RC_COMPILER windres.exe)
    endif()

    add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/zlib1rc.obj
                       COMMAND ${CMAKE_RC_COMPILER}
                            -D GCC_WINDRES
                            -I ${ZLIB_DIR}
                            -I ${CMAKE_CURRENT_BINARY_DIR}
                            -o ${CMAKE_CURRENT_BINARY_DIR}/zlib1rc.obj
                            -i ${ZLIB_DIR}/win32/zlib1.rc)
    set(ZLIB_DLL_SRCS ${CMAKE_CURRENT_BINARY_DIR}/zlib1rc.obj)
endif(MINGW)

add_library(zlib STATIC ${ZLIB_SRCS} ${ZLIB_ASMS} ${ZLIB_PUBLIC_HDRS} ${ZLIB_PRIVATE_HDRS})

CPM_ExportAdditionalIncludeDir("${ZLIB_DIR}")
CPM_ExportAdditionalLibraryTarget(zlib)
