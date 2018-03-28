cmake_minimum_required(VERSION 3.9)
project(boost CXX)

if(NOT CMAKE_SYSTEM_NAME)
    set(CMAKE_SYSTEM_NAME "${CMAKE_HOST_SYSTEM_NAME}")
endif()

if(NOT CMAKE_SYSTEM_PROCESSOR)
    set(CMAKE_SYSTEM_PROCESSOR ${CMAKE_HOST_SYSTEM_PROCESSOR})
endif()

find_path(ZLIB_INCLUDE zlib.h)
find_path(BZIP2_INCLUDE bzlib.h)
find_path(ICU_PATH include/unicode/utf.h)

# Add build type specific options
if(BUILD_SHARED_LIBS)
    list(APPEND B2_OPTIONS runtime-link=shared link=shared)
else()
    list(APPEND B2_OPTIONS runtime-link=static link=static)
endif()

if(CMAKE_SIZEOF_VOID_P STREQUAL 8)
    list(APPEND B2_OPTIONS address-model=64)
else()
    list(APPEND B2_OPTIONS address-model=32)
endif()

if(CMAKE_SYSTEM_PROCESSOR STREQUAL "AMD64" OR CMAKE_SYSTEM_PROCESSOR STREQUAL "x86_64")
    list(APPEND B2_OPTIONS architecture=x86)
elseif(CMAKE_SYSTEM_PROCESSOR STREQUAL "arm")
    list(APPEND B2_OPTIONS architecture=arm)
else()
    message(FATAL_ERROR "Unknown architecture: ${CMAKE_SYSTEM_PROCESSOR}")
endif()

if(CMAKE_SYSTEM_NAME STREQUAL "Darwin")
    list(APPEND B2_OPTIONS target-os=darwin toolset=gcc)
else()
    list(APPEND B2_OPTIONS target-os=linux toolset=gcc)
endif()

string(REPLACE " " " <compileflags>" CXXFLAGS "${CMAKE_CXX_FLAGS}")
#set(CXXFLAGS "${CXXFLAGS} <compileflags>-Wno-error=unused-command-line-argument")
if(CMAKE_CXX_COMPILER_TARGET)
    set(CXXFLAGS "${CXXFLAGS} <compileflags>${CMAKE_CXX_COMPILE_OPTIONS_TARGET} <compileflags>${CMAKE_CXX_COMPILER_TARGET}")
endif()
if(CMAKE_CXX_COMPILER_EXTERNAL_TOOLCHAIN)
    set(CXXFLAGS "${CXXFLAGS} <compileflags>${CMAKE_CXX_COMPILE_OPTIONS_EXTERNAL_TOOLCHAIN} <compileflags>${CMAKE_CXX_COMPILER_EXTERNAL_TOOLCHAIN}")
endif()
if(CMAKE_SYSROOT AND CMAKE_CXX_COMPILE_OPTIONS_SYSROOT)
    set(CXXFLAGS "${CXXFLAGS} <compileflags>${CMAKE_CXX_COMPILE_OPTIONS_SYSROOT}${CMAKE_SYSROOT}")
endif()
foreach(INCDIR ${CMAKE_CXX_STANDARD_INCLUDE_DIRECTORIES})
    set(CXXFLAGS "${CXXFLAGS} <compileflags>${CMAKE_INCLUDE_FLAG_C}${CMAKE_INCLUDE_FLAG_C_SEP}${INCDIR}")
endforeach()

find_library(ZLIB_LIBPATH z)
list(APPEND B2_OPTIONS
    -sZLIB_BINARY=z
    -sZLIB_LIBPATH="${ZLIB_LIBPATH}"
)
if(CMAKE_BUILD_TYPE STREQUAL "Release")
    find_library(BZIP2_LIBPATH bz2)
    list(APPEND B2_OPTIONS
        -sBZIP2_BINARY=bz2
        -sBZIP2_LIBPATH="${BZIP2_LIBPATH}"
        variant=release
    )
elseif(CMAKE_BUILD_TYPE STREQUAL "Debug")
    find_library(BZIP2_LIBPATH bz2d)
    list(APPEND B2_OPTIONS
        -sBZIP2_BINARY=bz2d
        -sBZIP2_LIBPATH="${BZIP2_LIBPATH}"
        variant=debug
    )
endif()

configure_file(${CMAKE_CURRENT_LIST_DIR}/user-config.jam ${CMAKE_CURRENT_BINARY_DIR}/user-config.jam @ONLY)

set(NUMBER_OF_PROCESSORS 1)
if(DEFINED ENV{NUMBER_OF_PROCESSORS})
    set(NUMBER_OF_PROCESSORS $ENV{NUMBER_OF_PROCESSORS})
elseif(CMAKE_HOST_LINUX)
    execute_process(
        COMMAND nproc
        OUTPUT_VARIABLE NUMBER_OF_PROCESSORS
    )
    string(REPLACE "\n" "" NUMBER_OF_PROCESSORS "${NUMBER_OF_PROCESSORS}")
    string(REPLACE " " "" NUMBER_OF_PROCESSORS "${NUMBER_OF_PROCESSORS}")
elseif(CMAKE_HOST_APPLE)
    execute_process(
        COMMAND sysctl -n hw.ncpu
        OUTPUT_VARIABLE NUMBER_OF_PROCESSORS
    )
    string(REPLACE "\n" "" NUMBER_OF_PROCESSORS "${NUMBER_OF_PROCESSORS}")
    string(REPLACE " " "" NUMBER_OF_PROCESSORS "${NUMBER_OF_PROCESSORS}")
endif()
if(NOT NUMBER_OF_PROCESSORS)
    set(NUMBER_OF_PROCESSORS 1)
endif()

add_custom_target(boost ALL
    COMMAND "${B2_EXE}"
        --user-config=${CMAKE_CURRENT_BINARY_DIR}/user-config.jam
        --stagedir=${CMAKE_CURRENT_BINARY_DIR}/stage
        --build-dir=${CMAKE_CURRENT_BINARY_DIR}
        ${B2_OPTIONS}
        --layout=system
        --with-atomic
        --with-random
        --with-date_time
        --with-filesystem
        --with-system
        --with-thread
        --with-chrono
        -sZLIB_INCLUDE="${ZLIB_INCLUDE}"
        -sBZIP2_INCLUDE="${BZIP2_INCLUDE}"
        -sICU_PATH="${ICU_PATH}"
        -j${NUMBER_OF_PROCESSORS}
        -sBOOST_ROOT=${BOOST_BUILD_PATH}
        -sBOOST_BUILD_PATH=${BOOST_BUILD_PATH}
        --debug-configuration
        --ignore-site-config
        --hash
        -q

        threading=multi
        threadapi=pthread
        debug-symbols=on
    WORKING_DIRECTORY ${SOURCE_PATH}
)

install(
    CODE "file(GLOB LIBS ${CMAKE_CURRENT_BINARY_DIR}/boost/build/*/*.a ${CMAKE_CURRENT_BINARY_DIR}/boost/build/*/*.so.* ${CMAKE_CURRENT_BINARY_DIR}/boost/build/*/*.dylib)\nif(LIBS)\nfile(INSTALL \${LIBS} DESTINATION \"\${CMAKE_INSTALL_PREFIX}/lib\")\nendif()"
)
