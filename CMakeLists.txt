
CMAKE_MINIMUM_REQUIRED(VERSION 3.23)

SET(NAME ppp)
PROJECT(${NAME} C CXX)

SET(PLATFORM_SYSTEM_LINUX FALSE)
SET(PLATFORM_SYSTEM_DARWIN FALSE)

IF(CMAKE_SYSTEM_NAME STREQUAL "Darwin" OR CMAKE_SYSTEM_NAME MATCHES "Darwin")
    SET(PLATFORM_SYSTEM_DARWIN TRUE)
ELSE()
    SET(PLATFORM_SYSTEM_LINUX TRUE)
ENDIF()

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

add_compile_options(
    -fPIE
    -fvisibility=hidden
    -Wno-format
    -Wno-deprecated-declarations
)

SET(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wno-implicit-function-declaration")

SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-inconsistent-missing-override -Wno-pointer-bool-conversion -Wno-unsequenced -Wno-null-dereference -Wno-gnu-variable-sized-type-not-at-end -Wno-tautological-undefined-compare -Wno-shift-count-overflow")

IF(CMAKE_BUILD_TYPE STREQUAL "Debug")
    SET(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -D_DEBUG -g3")
    SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -D_DEBUG -g3")
ELSE()
    SET(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -O3")
    SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -O3")
ENDIF()

ADD_DEFINITIONS(-DJEMALLOC)
ADD_DEFINITIONS(-DBUDDY_ALLOC_IMPLEMENTATION)
ADD_DEFINITIONS(-DBOOST_ERROR_CODE_HEADER_ONLY)
IF(PLATFORM_SYSTEM_LINUX)
    ADD_DEFINITIONS(-D_GNU_SOURCE)
ENDIF()


OPTION(PPP_FORCE_STATIC "Force static linking for libraries" ON)

IF(PPP_FORCE_STATIC)
    set(Boost_USE_STATIC_LIBS ON)
    set(Boost_USE_STATIC_RUNTIME OFF)
    set(OPENSSL_USE_STATIC_LIBS TRUE)
    
    SET(CMAKE_FIND_LIBRARY_SUFFIXES ".a")
    
    MESSAGE(STATUS "======== 启用了第三方依赖库的静态链接模式 ========")
ENDIF()

set(Boost_USE_MULTITHREADED ON)

find_package(Boost 1.70 REQUIRED COMPONENTS system coroutine thread context regex filesystem)


find_package(OpenSSL REQUIRED)

find_path(JEMALLOC_INCLUDE_DIR NAMES jemalloc/jemalloc.h)
find_library(JEMALLOC_LIBRARY NAMES libjemalloc.a jemalloc)

if(NOT JEMALLOC_LIBRARY)
    message(FATAL_ERROR "未找到 libjemalloc.a 静态库，请检查系统是否安装了 jemalloc 开发包")
endif()


INCLUDE_DIRECTORIES(
    ${PROJECT_SOURCE_DIR}/
    ${PROJECT_SOURCE_DIR}/common
    ${PROJECT_SOURCE_DIR}/common/json/include
    ${PROJECT_SOURCE_DIR}/common/lwip/my
    ${PROJECT_SOURCE_DIR}/common/lwip/include
    ${OPENSSL_INCLUDE_DIR}
    ${Boost_INCLUDE_DIRS}
    ${JEMALLOC_INCLUDE_DIR}
)


IF(PLATFORM_SYSTEM_DARWIN)
    FILE(GLOB_RECURSE PLATFORM_SOURCE_FILES
        ${PROJECT_SOURCE_DIR}/darwin/*.c
        ${PROJECT_SOURCE_DIR}/darwin/*.cpp)
ELSE()
    FILE(GLOB_RECURSE PLATFORM_SOURCE_FILES
        ${PROJECT_SOURCE_DIR}/linux/*.c
        ${PROJECT_SOURCE_DIR}/linux/*.cpp
        ${PROJECT_SOURCE_DIR}/ppp/tap/*.cpp
    )
ENDIF()

FILE(GLOB_RECURSE SOURCE_FILES
    ${PROJECT_SOURCE_DIR}/main.cpp
    ${PROJECT_SOURCE_DIR}/common/lwip/my/*.c
    ${PROJECT_SOURCE_DIR}/common/lwip/core/*.c
    ${PROJECT_SOURCE_DIR}/common/lwip/api/*.c
    ${PROJECT_SOURCE_DIR}/common/aggligator/*.cpp
    ${PROJECT_SOURCE_DIR}/common/base64/*.cpp
    ${PROJECT_SOURCE_DIR}/common/chnroutes2/*.cpp
    ${PROJECT_SOURCE_DIR}/common/json/src/*.cpp
    ${PROJECT_SOURCE_DIR}/common/libtcpip/*.cpp
    ${PROJECT_SOURCE_DIR}/common/dnslib/*.cpp
    ${PROJECT_SOURCE_DIR}/common/diagnostics/*.cpp
    ${PROJECT_SOURCE_DIR}/common/unix/*.c
    ${PROJECT_SOURCE_DIR}/common/unix/*.cpp
    ${PROJECT_SOURCE_DIR}/common/aesni/aes.cpp
    ${PROJECT_SOURCE_DIR}/ppp/*.c
    ${PROJECT_SOURCE_DIR}/ppp/*.cpp
)


ADD_EXECUTABLE(${NAME} ${SOURCE_FILES} ${PLATFORM_SOURCE_FILES})


SET(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR}/bin)

IF(NOT MSVC AND PPP_FORCE_STATIC)
    SET_TARGET_PROPERTIES(${NAME} PROPERTIES LINK_FLAGS "-static")
ENDIF()

set(CMAKE_THREAD_PREFER_PTHREAD TRUE)
find_package(Threads REQUIRED)

TARGET_LINK_LIBRARIES(${NAME}
    ${Boost_LIBRARIES}
    OpenSSL::SSL OpenSSL::Crypto
    ${JEMALLOC_LIBRARY}

    -static-libgcc
    -static-libstdc++

    ${CMAKE_THREAD_LIBS_INIT} # -lpthread (在 -static 下会自动寻找 libpthread.a)
    dl                        # libdl
    rt                        # librt

    lzma                      # 必须补在这里，解决 libunwind 的 lzma 依赖
    z                         # 补上 zlib 依赖

    -static
)
