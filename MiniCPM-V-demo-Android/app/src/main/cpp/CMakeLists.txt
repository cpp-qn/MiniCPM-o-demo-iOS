cmake_minimum_required(VERSION 3.22.1)

project("minicpm_v_demo" VERSION 1.0.0 LANGUAGES C CXX)

set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED true)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED true)

set(CMAKE_C_FLAGS   "${CMAKE_C_FLAGS}"   CACHE STRING "" FORCE)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}" CACHE STRING "" FORCE)

if(DEFINED ANDROID_ABI)
    message(STATUS "Detected Android ABI: ${ANDROID_ABI}")
    if(ANDROID_ABI STREQUAL "arm64-v8a")
        set(GGML_SYSTEM_ARCH "ARM")
        set(GGML_CPU_KLEIDIAI ON)
        set(GGML_OPENMP ON)
    elseif(ANDROID_ABI STREQUAL "x86_64")
        set(GGML_SYSTEM_ARCH "x86")
        set(GGML_CPU_KLEIDIAI OFF)
        set(GGML_OPENMP OFF)
    elseif(ANDROID_ABI STREQUAL "x86")
        set(GGML_SYSTEM_ARCH "x86")
        set(GGML_CPU_KLEIDIAI OFF)
        set(GGML_OPENMP OFF)
    else()
        message(WARNING "Unsupported ABI: ${ANDROID_ABI}, defaulting to generic CPU")
        set(GGML_SYSTEM_ARCH "")
        set(GGML_CPU_KLEIDIAI OFF)
        set(GGML_OPENMP OFF)
    endif()
endif()

# llama.cpp source lives at the repository root so it can be shared with the iOS demo.
# Allow override via -DLLAMA_SRC=... if you keep llama.cpp elsewhere.
if(NOT DEFINED LLAMA_SRC)
    set(LLAMA_SRC ${CMAKE_CURRENT_LIST_DIR}/../../../../../llama.cpp)
endif()
get_filename_component(LLAMA_SRC ${LLAMA_SRC} ABSOLUTE)
if(NOT EXISTS ${LLAMA_SRC}/CMakeLists.txt)
    message(FATAL_ERROR
        "llama.cpp not found at ${LLAMA_SRC}. "
        "Run `git submodule update --init` from the repository root.")
endif()
message(STATUS "Using llama.cpp source at: ${LLAMA_SRC}")

set(PATCH_DIR ${CMAKE_CURRENT_LIST_DIR}/patches)
set(SLICING_PATCH ${PATCH_DIR}/disable-slicing.patch)
if(EXISTS ${SLICING_PATCH})
    execute_process(
        COMMAND git -C ${LLAMA_SRC} apply --reverse --check ${SLICING_PATCH}
        RESULT_VARIABLE already_applied
        OUTPUT_QUIET ERROR_QUIET
    )
    if(already_applied EQUAL 0)
        message(STATUS "Patch already applied: disable-slicing.patch")
    else()
        execute_process(
            COMMAND git -C ${LLAMA_SRC} apply ${SLICING_PATCH}
            RESULT_VARIABLE patch_result
        )
        if(patch_result EQUAL 0)
            message(STATUS "Applied patch: disable-slicing.patch")
        else()
            message(FATAL_ERROR "Failed to apply patch: ${SLICING_PATCH}")
        endif()
    endif()
endif()

add_subdirectory(${LLAMA_SRC} ${CMAKE_BINARY_DIR}/build-llama)

if(NOT DEFINED LLAMA_INSTALL_VERSION)
    set(LLAMA_INSTALL_VERSION "0.0.0")
endif()

add_subdirectory(${LLAMA_SRC}/tools/mtmd ${CMAKE_BINARY_DIR}/build-mtmd)

add_library(${CMAKE_PROJECT_NAME} SHARED
        llama_jni.cpp)

target_compile_definitions(${CMAKE_PROJECT_NAME} PRIVATE
        GGML_SYSTEM_ARCH=${GGML_SYSTEM_ARCH}
        GGML_CPU_KLEIDIAI=$<BOOL:${GGML_CPU_KLEIDIAI}>
        GGML_OPENMP=$<BOOL:${GGML_OPENMP}>
)

target_include_directories(${CMAKE_PROJECT_NAME} PRIVATE
        ${LLAMA_SRC}
        ${LLAMA_SRC}/common
        ${LLAMA_SRC}/include
        ${LLAMA_SRC}/ggml/include
        ${LLAMA_SRC}/ggml/src
        ${LLAMA_SRC}/tools/mtmd)

target_link_libraries(${CMAKE_PROJECT_NAME}
        llama
        common
        mtmd
        android
        log)
