cmake_minimum_required(VERSION 3.10)

project(basicpitch.cpp)
enable_testing()

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

# --- Compiler Specific Flags ---
if(MSVC)
    # Windows-specific flags
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /W4")
    set(CMAKE_CXX_FLAGS_RELEASE "/O2 /fp:fast /GL /DNDEBUG")
else()
    # Linux and macOS (GCC/Clang)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -Wextra")
    set(CMAKE_CXX_FLAGS_RELEASE "-O3 -ffast-math -flto -fno-signed-zeros -fno-math-errno -DNDEBUG")
    
    # -march=native isn't supported on Apple Silicon (M1/M2/M3)
    if(NOT APPLE)
        set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -march=native")
    endif()
endif()

# --- Handle ONNX Runtime Path & Extension ---
if(APPLE)
    set(ORT_EXT "dylib")
    set(ORT_PLATFORM "macos")
elseif(WIN32)
    set(ORT_EXT "lib")
    set(ORT_PLATFORM "win")
else()
    set(ORT_EXT "so")
    set(ORT_PLATFORM "linux")
endif()

# Dynamically look for the library based on the OS
# Note: You may need to ensure your build folders match these naming conventions
set(ONNX_RUNTIME_LIB ${CMAKE_SOURCE_DIR}/../build/build-ort-${ORT_PLATFORM}/MinSizeRel/onnxruntime.${ORT_EXT})

# --- Includes ---
include_directories(
    ${CMAKE_CURRENT_SOURCE_DIR}/../vendor/onnxruntime/include
    ${CMAKE_CURRENT_SOURCE_DIR}/../ort-model/model
    ${CMAKE_CURRENT_SOURCE_DIR}/../src
    ${CMAKE_CURRENT_SOURCE_DIR}/../vendor/eigen
    ${CMAKE_CURRENT_SOURCE_DIR}/../vendor/oboe-resampler
)

# --- Source Discovery ---
file(GLOB SOURCES 
    "${CMAKE_CURRENT_SOURCE_DIR}/../src/*.cpp" 
    "${CMAKE_CURRENT_SOURCE_DIR}/../src_cli/*.cpp" 
    "${CMAKE_CURRENT_SOURCE_DIR}/../ort-model/model/model.ort.c" 
    "${CMAKE_CURRENT_SOURCE_DIR}/../vendor/oboe-resampler/*.cpp"
)

add_executable(basicpitch ${SOURCES})

# --- Target Specifics ---
target_include_directories(basicpitch SYSTEM PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/../vendor/libremidi/include)
target_compile_definitions(basicpitch PRIVATE LIBREMIDI_HEADER_ONLY=1)
target_link_libraries(basicpitch ${ONNX_RUNTIME_LIB} libnyquist)

# --- Cross-platform Linting ---
file(GLOB SOURCES_TO_LINT 
    "${CMAKE_CURRENT_SOURCE_DIR}/../src/*.cpp" 
    "${CMAKE_CURRENT_SOURCE_DIR}/../src/*.hpp" 
    "${CMAKE_CURRENT_SOURCE_DIR}/../src_wasm/*.cpp" 
    "${CMAKE_CURRENT_SOURCE_DIR}/../src_cli/*.cpp"
)

# Only define lint target if not on Windows (as these tools are usually Unix-based)
if(NOT WIN32)
    add_custom_target(lint
        COMMAND clang-format -i ${SOURCES_TO_LINT} --style=file
        COMMAND cppcheck -I"src/" -I"cli-apps/" --enable=all --suppress=missingIncludeSystem ${SOURCES_TO_LINT} --std=c++17
        WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
    )
endif()