cmake_minimum_required(VERSION 3.0)
project(VulkanSift DESCRIPTION "SIFT GPU implementation using Vulkan" LANGUAGES C CXX)


#####################################################################################"
# Set default value for build type to Release
if(NOT CMAKE_BUILD_TYPE)
    message("CMAKE_BUILD_TYPE not specified. Build type will be set to Release.")
    set(CMAKE_BUILD_TYPE Release)
endif()

#####################################################################################
# Find the Vulkan SDK
if(NOT CMAKE_VERSION VERSION_LESS 3.7.0) #if FindCmake is available
    find_package(Vulkan REQUIRED)
elseif(DEFINED ENV{VULKAN_SDK}) # if VULKAN_SDK in the path
    set(Vulkan_INCLUDE_DIR "$ENV{VULKAN_SDK}/lib")
    find_library(Vulkan_LIBRARY NAMES vulkan-1 vulkan PATHS "$ENV{VULKAN_SDK}/lib")
else()
    message(FATAL_ERROR "Could not find the Vulkan SDK")
endif()

#####################################################################################"
# Define project options and variables
option(BUILD_SHARED_LIBS "Use to build shared library" ON)
option(VULKANSIFT_LOAD_VK_AT_RUNTIME "Use to dynamically load the Vulkan library at runtime" ON)
option(VULKANSIFT_BUILD_EXAMPLES "Use to build VulkanSift examples." OFF)
option(VULKANSIFT_WITH_GPU_DEBUG_EXAMPLE "Use to build an example that can be used with GPU profilers/debuggers" OFF)
option(VULKANSIFT_BUILD_PERFS "Use to build VulkanSift performance tests." OFF)
set(VULKANSIFT_SANITIZE "" CACHE STRING "Use with value \"Address\" to build using the address sanitizer or 
    with \"Thread\" to build using the thread sanitizer. Leave empty otherwise.")

#####################################################################################"
# Check if compiler is Clang or GNU (avoids lots of trouble with optimization related parameters)
if (NOT CMAKE_CXX_COMPILER_ID MATCHES "Clang|GNU|MSVC")
     message(FATAL_ERROR "Only GNU, Clang and MSVC compilers are supported in this project.")
endif()

# Check if platform is Linux or Windows (would need MoltenVK for Mac)
if (NOT CMAKE_SYSTEM_NAME MATCHES "Linux|Windows|Android")
    message(FATAL_ERROR "Only Linux, Windows and Android platforms are supported in this project.")
endif()
#####################################################################################
# VulkanSift sources
set(VULKANSIFT_LIB_SRC
    src/vulkansift/vulkansift.c
    src/vulkansift/sift_memory.c
    src/vulkansift/sift_detector.c
    src/vulkansift/sift_matcher.c
    # vkenv files (currently from git submodule)
    src/vulkansift/vkenv/vulkan_loader.c
    src/vulkansift/vkenv/vulkan_device.c
    src/vulkansift/vkenv/vulkan_utils.c
    src/vulkansift/vkenv/vulkan_swapchain.c
    src/vulkansift/vkenv/vulkan_surface.c
    src/vulkansift/vkenv/debug_presenter.c
    src/vulkansift/vkenv/logger.c
    # we use embedded shaders we need to add the source storing the spv to the library
    ${CMAKE_CURRENT_BINARY_DIR}/generated/embedded_shader_provider.c
)

# VulkanSift shaders
set(VULKANSIFT_LIB_SHADERS
    # SIFT detector algorithm shaders
    ${CMAKE_CURRENT_SOURCE_DIR}/src/vulkansift/shaders/GaussianBlur.comp
    ${CMAKE_CURRENT_SOURCE_DIR}/src/vulkansift/shaders/GaussianBlurInterpolated.comp
    ${CMAKE_CURRENT_SOURCE_DIR}/src/vulkansift/shaders/DifferenceOfGaussian.comp
    ${CMAKE_CURRENT_SOURCE_DIR}/src/vulkansift/shaders/ExtractKeypoints.comp
    ${CMAKE_CURRENT_SOURCE_DIR}/src/vulkansift/shaders/ComputeOrientation.comp
    ${CMAKE_CURRENT_SOURCE_DIR}/src/vulkansift/shaders/ComputeDescriptors.comp
    # SIFT matching algorithm shaders
    ${CMAKE_CURRENT_SOURCE_DIR}/src/vulkansift/shaders/Get2NearestNeighbors.comp
)

#####################################################################################
#####################################################################################
# Handle CXX compiler and linker options
set(VULKANSIFT_COMPILER_OPTIONS "-Wall;-O3;$<$<CONFIG:Debug>:-g>;-fvisibility=hidden;")
if (NOT CMAKE_CXX_COMPILER_ID MATCHES "MSVC")
    string(APPEND VULKANSIFT_COMPILER_OPTIONS "-Wextra;")
endif()
set(VULKANSIFT_LINKER_OPTIONS "")

#####################################################################################
# Add options for address and thread sanitizers (debug only) (may not be supported by every compiler)
# Use -DSANITIZE:BOOL=Address or -DSANITIZE:BOOL=Thread to activate with -DCMAKE_BUILD_TYPE:String=Debug
if(VULKANSIFT_SANITIZE STREQUAL Address)
    string(APPEND VULKANSIFT_COMPILER_OPTIONS "-fsanitize=address")
    string(APPEND VULKANSIFT_LINKER_OPTIONS "-fsanitize=address")
elseif(VULKANSIFT_SANITIZE STREQUAL Thread)
    string(APPEND VULKANSIFT_COMPILER_OPTIONS "-fsanitize=thread;")
    string(APPEND VULKANSIFT_LINKER_OPTIONS "-fsanitize=thread")
endif()
#####################################################################################
#####################################################################################
# Build the VulkanSift library
add_library(vulkansift ${VULKANSIFT_LIB_SRC})
target_compile_options(vulkansift PRIVATE "${VULKANSIFT_COMPILER_OPTIONS}")
set_property(TARGET vulkansift APPEND_STRING PROPERTY LINK_FLAGS " ${VULKANSIFT_LINKER_OPTIONS}")

# Provides access to the interface files for any project using vksift inside their Cmake build
# using add_subdirectory() and target_link_libraries(target_name vulkansift ...)
target_include_directories(vulkansift PUBLIC $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/include/> $<INSTALL_INTERFACE:include>)

if(NOT ${CMAKE_PROJECT_NAME} STREQUAL ${PROJECT_NAME})
    # Only if there's a parent CMake project
    # Set the VulkanSift_LIB and VulkanSift_INCLUDE_DIR variable in the parent scope for compatibility with installed versions
    set(VulkanSift_LIB vulkansift PARENT_SCOPE)
    set(VulkanSift_INCLUDE_DIR ${PROJECT_SOURCE_DIR}/include/ PARENT_SCOPE)
endif()

# Private include directories used when building the library
target_include_directories(vulkansift PRIVATE ${PROJECT_SOURCE_DIR}/src/)
target_include_directories(vulkansift PRIVATE ${Vulkan_INCLUDE_DIRS})

if(CMAKE_SYSTEM_NAME MATCHES "Android")
    # Add the Android log library to send logs to adb
    find_library(android-log-lib log REQUIRED)
    target_link_libraries(vulkansift ${android-log-lib})
endif()

if(VULKANSIFT_LOAD_VK_AT_RUNTIME)
    target_compile_definitions(vulkansift PRIVATE VK_NO_PROTOTYPES=1)
    if(NOT CMAKE_SYSTEM_NAME MATCHES "Windows")
        target_link_libraries(vulkansift dl)
    endif()
else()
    target_link_libraries(vulkansift  ${Vulkan_LIBRARY})
endif()

# Set Vulkan defines to load platform specific functions (required to use surface creation related functions)
if(CMAKE_SYSTEM_NAME MATCHES "Linux")
    # Only Xlib is supported with Linux for now
    target_compile_definitions(vulkansift PRIVATE VK_USE_PLATFORM_XLIB_KHR=1)
elseif(CMAKE_SYSTEM_NAME MATCHES "Windows")
    target_compile_definitions(vulkansift PRIVATE VK_USE_PLATFORM_WIN32_KHR=1)
elseif(CMAKE_SYSTEM_NAME MATCHES "Android")
    target_compile_definitions(vulkansift PRIVATE VK_USE_PLATFORM_ANDROID_KHR=1)
endif()

# Add dependencies on shader files
include("cmake/ShaderUtils.cmake")
addEmbeddedSpvShaderDependency(vulkansift "${VULKANSIFT_LIB_SHADERS}" ${CMAKE_CURRENT_BINARY_DIR}/generated/embedded_shader_provider.c)
target_compile_definitions(vulkansift PRIVATE VKENV_USE_EMBEDDED_SHADERS=1)

#####################################################################################
##########################################################################
# Example and perf executables

if(VULKANSIFT_BUILD_EXAMPLES)
message(STATUS "Examples will be built")
find_package(OpenCV REQUIRED core highgui)
# Example program that read an image, detect and draw the SIFT features
add_executable(test_sift_detect src/examples/test_sift_detect.cpp src/examples/test_utils.cpp)
target_link_libraries(test_sift_detect vulkansift ${OpenCV_LIBS})
target_compile_options(test_sift_detect PRIVATE "${VULKANSIFT_COMPILER_OPTIONS}")
set_property(TARGET test_sift_detect APPEND_STRING PROPERTY LINK_FLAGS " ${VULKANSIFT_LINKER_OPTIONS}")

# Example program that read two images and perform SIFT feature matching
add_executable(test_sift_match src/examples/test_sift_match.cpp src/examples/test_utils.cpp)
target_link_libraries(test_sift_match vulkansift ${OpenCV_LIBS})
target_compile_options(test_sift_match PRIVATE "${VULKANSIFT_COMPILER_OPTIONS}")
set_property(TARGET test_sift_match APPEND_STRING PROPERTY LINK_FLAGS " ${VULKANSIFT_LINKER_OPTIONS}")

# Example program that detect SIFT features and show the scale-space pyramid images
add_executable(test_sift_show_pyr src/examples/test_sift_show_pyr.cpp src/examples/test_utils.cpp)
target_link_libraries(test_sift_show_pyr vulkansift ${OpenCV_LIBS})
target_compile_options(test_sift_show_pyr PRIVATE "${VULKANSIFT_COMPILER_OPTIONS}")
set_property(TARGET test_sift_show_pyr APPEND_STRING PROPERTY LINK_FLAGS " ${VULKANSIFT_LINKER_OPTIONS}")

# Example program that shows how to use C++ exception based error handling to handle unexpected errors
add_executable(test_sift_error_handling src/examples/test_sift_error_handling.cpp)
target_link_libraries(test_sift_error_handling vulkansift)
target_compile_options(test_sift_error_handling PRIVATE "${VULKANSIFT_COMPILER_OPTIONS}")
set_property(TARGET test_sift_error_handling APPEND_STRING PROPERTY LINK_FLAGS " ${VULKANSIFT_LINKER_OPTIONS}")

if(VULKANSIFT_WITH_GPU_DEBUG_EXAMPLE AND NOT CMAKE_SYSTEM_NAME MATCHES "Android")
    find_package(glfw3 REQUIRED)
    # Example program that enables GPU debug/profiling with external tools (NVIDIA Nsight, Renderdoc)
    add_executable(test_sift_gpu_debug src/examples/test_sift_gpu_debug.cpp src/examples/test_utils.cpp)
    target_link_libraries(test_sift_gpu_debug vulkansift ${OpenCV_LIBS} glfw)
    target_compile_options(test_sift_gpu_debug PRIVATE "${VULKANSIFT_COMPILER_OPTIONS}")
    set_property(TARGET test_sift_gpu_debug APPEND_STRING PROPERTY LINK_FLAGS " ${VULKANSIFT_LINKER_OPTIONS}")
endif()

endif(VULKANSIFT_BUILD_EXAMPLES)

if(VULKANSIFT_BUILD_PERFS)
message(STATUS "Performance tests will be built")
# Add perf executables
find_package(OpenCV REQUIRED core highgui features2d)

# Appart from OpenCV, all compared SIFT implementations are compiled separately.
# Their headers are placed in external_libs/SIFT_IMPL_NAME/src/ and the library placed in external_libs/SIFT_IMPL_NAME/lib/.
set(VULKAN_SIFT_PERFS_COMMON_SRC 
    src/perf/perf_common.cpp
    src/perf/wrappers/vulkansift_wrapper.cpp
    src/perf/wrappers/opencv_wrapper.cpp
    src/perf/wrappers/vlfeat_wrapper.cpp
    src/perf/wrappers/siftgpu_wrapper.cpp
    src/perf/wrappers/popsift_wrapper.cpp
)
set(VULKAN_SIFT_PERFS_COMMON_INC
    $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/src/>
    $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/external_libs/>
)
set(VULKAN_SIFT_PERFS_COMMON_LIBS
    vulkansift
    pthread
    omp
    GL
    GLEW
    cuda
    cudart
    X11
    ${OpenCV_LIBS}
    ${CMAKE_CURRENT_SOURCE_DIR}/external_libs/vlfeat/lib/libvlfeat.a
    ${CMAKE_CURRENT_SOURCE_DIR}/external_libs/SiftGPU/lib/libsift_gpu.a
    ${CMAKE_CURRENT_SOURCE_DIR}/external_libs/popsift/lib/libpopsift.so
)
add_executable(perf_sift_match src/perf/perf_matching.cpp ${VULKAN_SIFT_PERFS_COMMON_SRC})
target_compile_options(perf_sift_match PRIVATE "-O3;-fopenmp")
target_include_directories(perf_sift_match PRIVATE ${VULKAN_SIFT_PERFS_COMMON_INC})
target_link_libraries(perf_sift_match ${VULKAN_SIFT_PERFS_COMMON_LIBS})
target_compile_options(perf_sift_match PRIVATE "${VULKANSIFT_COMPILER_OPTIONS}")
set_property(TARGET perf_sift_match APPEND_STRING PROPERTY LINK_FLAGS " ${VULKANSIFT_LINKER_OPTIONS}")

add_executable(perf_sift_runtime src/perf/perf_runtime.cpp ${VULKAN_SIFT_PERFS_COMMON_SRC})
target_compile_options(perf_sift_runtime PRIVATE "-O3;-fopenmp")
target_include_directories(perf_sift_runtime PRIVATE ${VULKAN_SIFT_PERFS_COMMON_INC})
target_link_libraries(perf_sift_runtime ${VULKAN_SIFT_PERFS_COMMON_LIBS})
target_compile_options(perf_sift_runtime PRIVATE "${VULKANSIFT_COMPILER_OPTIONS}")
set_property(TARGET perf_sift_runtime APPEND_STRING PROPERTY LINK_FLAGS " ${VULKANSIFT_LINKER_OPTIONS}")

endif()

##########################################################################
##########################################################################
# Install library
set(VULKAN_SIFT_TARGETS vulkansift)

# Copy include directory content to INSTALL include dir
install(DIRECTORY ${PROJECT_SOURCE_DIR}/include/ DESTINATION include)

install(TARGETS vulkansift
    EXPORT VulkanSiftTargets
    LIBRARY DESTINATION lib
    ARCHIVE DESTINATION lib
    RUNTIME DESTINATION bin
    PUBLIC_HEADER DESTINATION include
)

# Use CMakePackageConfigHelpers to define Cmake config files to install
# This allows users to use the library using find_package(VulkanSift)
# and target_link_libraries(target_name ${VulkanSift_LIB})
include(CMakePackageConfigHelpers)
write_basic_package_version_file(
    "${CMAKE_CURRENT_BINARY_DIR}/VulkanSiftConfigVersion.cmake"
    VERSION 0.1
    COMPATIBILITY AnyNewerVersion
)
configure_package_config_file(
    "${PROJECT_SOURCE_DIR}/cmake/VulkanSiftConfig.cmake.in"
    "${PROJECT_BINARY_DIR}/VulkanSiftConfig.cmake"
    INSTALL_DESTINATION lib/cmake/VulkanSift
)

install(EXPORT VulkanSiftTargets DESTINATION lib/cmake/VulkanSift)
install(FILES
    "${PROJECT_BINARY_DIR}/VulkanSiftConfigVersion.cmake"
    "${PROJECT_BINARY_DIR}/VulkanSiftConfig.cmake"
    DESTINATION lib/cmake/VulkanSift)

