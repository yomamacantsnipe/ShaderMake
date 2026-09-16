cmake_minimum_required(VERSION 3.15)

project(ShaderMake LANGUAGES C CXX)

# Globals?
set_property(GLOBAL PROPERTY USE_FOLDERS ON)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_C_STANDARD 99)

# Compile options
if(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
    set(COMPILE_OPTIONS -Wextra)
elseif(CMAKE_CXX_COMPILER_ID MATCHES "GNU")
    set(COMPILE_OPTIONS -Wextra)
elseif(MSVC)
    set(COMPILE_OPTIONS /W4 /WX)
else()
    message(WARNING "ShaderMake: Unknown compiler!")
endif()

# ShaderMake
add_executable(${PROJECT_NAME}
    "argparse.h"
    "argparse.c"
    "ShaderBlob.h"
    "ShaderBlob.cpp"
    "ShaderMake.cpp"
)
target_compile_options(${PROJECT_NAME} PRIVATE ${COMPILE_OPTIONS})

if(SHADERMAKE_BIN_OUTPUT_PATH)
    set_target_properties(${PROJECT_NAME} PROPERTIES
        RUNTIME_OUTPUT_DIRECTORY "${SHADERMAKE_BIN_OUTPUT_PATH}/$<CONFIG>"
        RUNTIME_OUTPUT_DIRECTORY_DEBUG "${SHADERMAKE_BIN_OUTPUT_PATH}/Debug"
        RUNTIME_OUTPUT_DIRECTORY_MINSIZEREL "${SHADERMAKE_BIN_OUTPUT_PATH}/MinSizeRel"
        RUNTIME_OUTPUT_DIRECTORY_RELEASE "${SHADERMAKE_BIN_OUTPUT_PATH}/Release"
        RUNTIME_OUTPUT_DIRECTORY_RELWITHDEBINFO "${SHADERMAKE_BIN_OUTPUT_PATH}/RelWithDebInfo"
    )
endif()

set_property(TARGET ${PROJECT_NAME} PROPERTY FOLDER "ShaderMake")

if(WIN32)
    target_compile_definitions(${PROJECT_NAME} PRIVATE
        WIN32_LEAN_AND_MEAN
        NOMINMAX
        _CRT_SECURE_NO_WARNINGS
    )
elseif(APPLE)
    target_link_libraries(${PROJECT_NAME} pthread)
else()
    target_link_libraries(${PROJECT_NAME} stdc++fs pthread)
endif()

# Optional spirv-cross for the MSL platform (HLSL -> SPIR-V -> MSL in-process).
option(SHADERMAKE_WITH_SPIRV_CROSS "Enable MSL output via spirv-cross" ON)
if(SHADERMAKE_WITH_SPIRV_CROSS)
    find_path(SPIRV_CROSS_INCLUDE_DIR spirv_cross/spirv_msl.hpp
        PATHS /opt/local/include /usr/local/include /opt/homebrew/include)
    find_library(SPIRV_CROSS_MSL_LIB spirv-cross-msl PATHS /opt/local/lib /usr/local/lib /opt/homebrew/lib)
    find_library(SPIRV_CROSS_GLSL_LIB spirv-cross-glsl PATHS /opt/local/lib /usr/local/lib /opt/homebrew/lib)
    find_library(SPIRV_CROSS_CORE_LIB spirv-cross-core PATHS /opt/local/lib /usr/local/lib /opt/homebrew/lib)
    find_library(SPIRV_CROSS_UTIL_LIB spirv-cross-util PATHS /opt/local/lib /usr/local/lib /opt/homebrew/lib)
    if(SPIRV_CROSS_INCLUDE_DIR AND SPIRV_CROSS_MSL_LIB AND SPIRV_CROSS_GLSL_LIB AND SPIRV_CROSS_CORE_LIB)
        target_compile_definitions(${PROJECT_NAME} PRIVATE SHADERMAKE_WITH_SPIRV_CROSS=1)
        target_include_directories(${PROJECT_NAME} PRIVATE ${SPIRV_CROSS_INCLUDE_DIR})
        # Link order matters for static libs: msl -> glsl -> core -> util
        target_link_libraries(${PROJECT_NAME}
            ${SPIRV_CROSS_MSL_LIB} ${SPIRV_CROSS_GLSL_LIB} ${SPIRV_CROSS_CORE_LIB})
        if(SPIRV_CROSS_UTIL_LIB)
            target_link_libraries(${PROJECT_NAME} ${SPIRV_CROSS_UTIL_LIB})
        endif()
        message(STATUS "ShaderMake: MSL output enabled (spirv-cross found)")
    else()
        message(STATUS "ShaderMake: spirv-cross not found, MSL output disabled")
    endif()
endif()
