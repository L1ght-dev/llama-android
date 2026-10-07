cmake_minimum_required(VERSION 3.22.1)
project(llama_android LANGUAGES C CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# --- 16 KB page alignment (Android 15 ready) -------------------------------
add_link_options(
    "-Wl,-z,max-page-size=16384"
    "-Wl,-z,common-page-size=16384")

# --- Fetch SPIRV-Headers تلقائياً لتشغيل معالج الرسوميات Vulkan -------------
include(FetchContent)
FetchContent_Declare(
    spirv_headers
    GIT_REPOSITORY https://github.com/KhronosGroup/SPIRV-Headers.git
    GIT_TAG        main
)
FetchContent_MakeAvailable(spirv_headers)

# المسار الصحيح المباشر لملف التكوين
set(SPIRV-Headers_DIR "${spirv_headers_BINARY_DIR}" CACHE PATH "" FORCE)
list(APPEND CMAKE_PREFIX_PATH "${spirv_headers_BINARY_DIR}")

# --- llama.cpp submodule at repo root --------------------------------------
set(LLAMA_DIR ${CMAKE_CURRENT_SOURCE_DIR}/../../../../llama.cpp)

# Enable Vulkan GPU acceleration
set(BUILD_SHARED_LIBS    ON  CACHE BOOL "" FORCE)
set(GGML_OPENMP          OFF CACHE BOOL "" FORCE)
set(GGML_VULKAN          ON  CACHE BOOL "" FORCE)
set(GGML_LLAMAFILE       OFF CACHE BOOL "" FORCE)
set(LLAMA_CURL           OFF CACHE BOOL "" FORCE)
set(LLAMA_BUILD_TESTS    OFF CACHE BOOL "" FORCE)
set(LLAMA_BUILD_EXAMPLES OFF CACHE BOOL "" FORCE)
set(LLAMA_BUILD_SERVER   OFF CACHE BOOL "" FORCE)
set(LLAMA_BUILD_TOOLS    OFF CACHE BOOL "" FORCE)
set(LLAMA_BUILD_COMMON   OFF CACHE BOOL "" FORCE)
add_subdirectory(${LLAMA_DIR} ${CMAKE_CURRENT_BINARY_DIR}/llama.cpp)

# --- JNI bridge → libllama_jni.so ------------------------------------------
add_library(llama_jni SHARED llama_jni.cpp)
find_library(log-lib log)
find_library(vulkan-lib vulkan)

target_include_directories(llama_jni PRIVATE
    ${LLAMA_DIR}/include
    ${LLAMA_DIR}/ggml/include)

target_link_libraries(llama_jni
    llama
    ${log-lib}
    ${vulkan-lib})
