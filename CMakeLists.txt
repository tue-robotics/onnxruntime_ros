cmake_minimum_required(VERSION 3.24 FATAL_ERROR)
project(onnxruntime_ros)

find_package(ament_cmake REQUIRED)

# The package version is the ONNX Runtime version that gets downloaded
ament_package_xml()
set(ONNXRUNTIME_VERSION "${${PROJECT_NAME}_VERSION}")

if(NOT CMAKE_SYSTEM_NAME STREQUAL "Linux" OR NOT CMAKE_SYSTEM_PROCESSOR MATCHES "^(x86_64|amd64|AMD64)$")
  message(FATAL_ERROR "The ONNX Runtime binary archive requires Linux x86_64")
endif()

include(CheckLanguage)
check_language(CUDA)
if(CMAKE_CUDA_COMPILER)
  enable_language(CUDA)
  message(STATUS "Using the ONNX Runtime GPU (CUDA/TensorRT) build")
  set(${PROJECT_NAME}_CUDA_ENABLED TRUE)
  set(ONNXRUNTIME_FILENAME "onnxruntime-linux-x64-gpu-${ONNXRUNTIME_VERSION}.tgz")
else()
  message(STATUS "Using the ONNX Runtime CPU build")
  set(${PROJECT_NAME}_CUDA_ENABLED FALSE)
  set(ONNXRUNTIME_FILENAME "onnxruntime-linux-x64-${ONNXRUNTIME_VERSION}.tgz")
endif()

set(ONNXRUNTIME_URL_BASE "https://github.com/microsoft/onnxruntime/releases/download/v${ONNXRUNTIME_VERSION}")

set(EP_NAME "EP_${PROJECT_NAME}")
set(ONNXRUNTIME_SOURCE_DIR "${CMAKE_CURRENT_BINARY_DIR}/${EP_NAME}-prefix/src/${EP_NAME}")

include(ExternalProject)
externalproject_add(${EP_NAME}
  URL "${ONNXRUNTIME_URL_BASE}/${ONNXRUNTIME_FILENAME}"
  DOWNLOAD_EXTRACT_TIMESTAMP TRUE
  DOWNLOAD_NO_PROGRESS TRUE
  CONFIGURE_COMMAND ""
  BUILD_COMMAND ""
  UPDATE_COMMAND ""
  INSTALL_COMMAND ""
)

# Headers go in a package-specific directory, so they don't end up directly in a merged include directory
install(DIRECTORY "${ONNXRUNTIME_SOURCE_DIR}/include/" DESTINATION include/${PROJECT_NAME})
# Preserve the archive's SONAME symlinks and install all runtime libraries, including the
# execution providers (shared, CUDA, TensorRT) that libonnxruntime loads at runtime.
# The archive's CMake config and pkg-config files are skipped, they point to the wrong directories.
install(DIRECTORY "${ONNXRUNTIME_SOURCE_DIR}/lib/" DESTINATION lib
  USE_SOURCE_PERMISSIONS
  FILES_MATCHING PATTERN "*.so*"
  PATTERN "cmake" EXCLUDE
  PATTERN "pkgconfig" EXCLUDE)

if(BUILD_TESTING)
  find_package(ament_cmake_lint_cmake REQUIRED)
  find_package(ament_cmake_xmllint REQUIRED)
  ament_lint_cmake(MAX_LINE_LENGTH 120)
  ament_xmllint(MAX_LINE_LENGTH 120)
endif()

ament_package(CONFIG_EXTRAS "cmake/${PROJECT_NAME}-extras.cmake.in")
