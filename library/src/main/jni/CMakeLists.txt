cmake_minimum_required(VERSION 3.22.1)
project(llama_android LANGUAGES C CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# --- 16 KB page alignment (Android 15 ready) -------------------------------
add_link_options(
    "-Wl,-z,max-page-size=16384"
    "-Wl,-z,common-page-size=16384")

# --- Fetch SPIRV-Headers ---------------------------------------------------
include(FetchContent)
FetchContent_Declare(
    spirv_headers
    GIT_REPOSITORY https://github.com/KhronosGroup/SPIRV-Headers.git
    GIT_TAG        main
    GIT_SHALLOW    TRUE
)
FetchContent_GetProperties(spirv_headers)
if(NOT spirv_headers_POPULATED)
    FetchContent_Populate(spirv_headers)
endif()

file(MAKE_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/spirv_headers_config")
file(WRITE "${CMAKE_CURRENT_BINARY_DIR}/spirv_headers_config/SPIRV-HeadersConfig.cmake"
"set(SPIRV-Headers_FOUND TRUE)
set(SPIRV_HEADERS_FOUND TRUE)
set(SPIRV-Headers_INCLUDE_DIRS \"${spirv_headers_SOURCE_DIR}/include\")
set(SPIRV_HEADERS_INCLUDE_DIRS \"${spirv_headers_SOURCE_DIR}/include\")
if(NOT TARGET SPIRV-Headers::SPIRV-Headers)
    add_library(SPIRV-Headers::SPIRV-Headers INTERFACE IMPORTED)
    set_target_properties(SPIRV-Headers::SPIRV-Headers PROPERTIES
        INTERFACE_INCLUDE_DIRECTORIES \"${spirv_headers_SOURCE_DIR}/include\"
    )
endif()
")
set(SPIRV-Headers_DIR "${CMAKE_CURRENT_BINARY_DIR}/spirv_headers_config" CACHE PATH "" FORCE)
list(APPEND CMAKE_PREFIX_PATH "${CMAKE_CURRENT_BINARY_DIR}/spirv_headers_config")

# --- Fetch Vulkan-Hpp ------------------------------------------------------
FetchContent_Declare(
    vulkan_hpp
    GIT_REPOSITORY https://github.com/KhronosGroup/Vulkan-Hpp.git
    GIT_TAG        v1.3.275
    GIT_SHALLOW    TRUE
)
FetchContent_GetProperties(vulkan_hpp)
if(NOT vulkan_hpp_POPULATED)
    FetchContent_Populate(vulkan_hpp)
endif()

include_directories(${spirv_headers_SOURCE_DIR}/include)
include_directories(${vulkan_hpp_SOURCE_DIR})
include_directories(${vulkan_hpp_SOURCE_DIR}/include)
add_compile_definitions(VULKAN_HPP_NO_VERSION_CHECK)
add_compile_definitions(GGML_USE_VULKAN)

# --- llama.cpp configuration ----------------------------------------------
set(LLAMA_DIR ${CMAKE_CURRENT_SOURCE_DIR}/../../../../llama.cpp)

set(BUILD_SHARED_LIBS    ON  CACHE BOOL "" FORCE)
set(GGML_OPENMP          OFF CACHE BOOL "" FORCE)
set(GGML_VULKAN          ON  CACHE BOOL "" FORCE)
# تعطيل البحث الديناميكي الفاشل في أندرويد لدمج Vulkan مباشرة
set(GGML_BACKEND_DL      OFF CACHE BOOL "" FORCE)
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
    ${LLAMA_DIR}/ggml/include
    ${spirv_headers_SOURCE_DIR}/include
    ${vulkan_hpp_SOURCE_DIR}
    ${vulkan_hpp_SOURCE_DIR}/include)

# ربط ggml-vulkan مباشرة بـ llama_jni لإجبار أندرويد على تحميله
target_link_libraries(llama_jni
    llama
    ggml-vulkan
    ${log-lib}
    ${vulkan-lib})
