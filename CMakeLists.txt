cmake_minimum_required(VERSION 3.24 FATAL_ERROR)
project(onnxruntime_ros)

find_package(ament_cmake REQUIRED)

# The package version is the ONNX Runtime version that gets downloaded
ament_package_xml()

if(NOT CMAKE_SYSTEM_NAME STREQUAL "Linux" OR NOT CMAKE_SYSTEM_PROCESSOR MATCHES "^(x86_64|amd64|AMD64)$")
  message(FATAL_ERROR "The ONNX Runtime binary archive requires Linux x86_64")
endif()

include(CheckLanguage)
check_language(CUDA)
if(CMAKE_CUDA_COMPILER)
  enable_language(CUDA)
  message(STATUS "Using the ONNX Runtime GPU (CUDA/TensorRT) build")
  set(${PROJECT_NAME}_CUDA_ENABLED TRUE)
  set(${PROJECT_NAME}_ARCHIVE_NAME "onnxruntime-linux-x64-gpu-${${PROJECT_NAME}_VERSION}.tgz")
else()
  message(STATUS "Using the ONNX Runtime CPU build")
  set(${PROJECT_NAME}_CUDA_ENABLED FALSE)
  set(${PROJECT_NAME}_ARCHIVE_NAME "onnxruntime-linux-x64-${${PROJECT_NAME}_VERSION}.tgz")
endif()

set(${PROJECT_NAME}_URL_BASE "https://github.com/microsoft/onnxruntime/releases/download/v${${PROJECT_NAME}_VERSION}")
set(${PROJECT_NAME}_URL "${${PROJECT_NAME}_URL_BASE}/${${PROJECT_NAME}_ARCHIVE_NAME}")

# Download at configure time instead of via externalproject_add, so a GITHUB_TOKEN is only kept in memory.
# externalproject_add would write its HTTP_HEADER to the generated download script in the build directory.
set(${PROJECT_NAME}_ARCHIVE "${CMAKE_CURRENT_BINARY_DIR}/${${PROJECT_NAME}_ARCHIVE_NAME}")
if(NOT EXISTS "${${PROJECT_NAME}_ARCHIVE}")
  # Authenticate the download when a GITHUB_TOKEN is available, to avoid rate limiting
  set(_download_args)
  if(NOT "$ENV{GITHUB_TOKEN}" STREQUAL "")
    list(APPEND _download_args HTTPHEADER "Authorization: Bearer $ENV{GITHUB_TOKEN}")
  endif()
  message(STATUS "Downloading ${${PROJECT_NAME}_URL}")
  file(DOWNLOAD "${${PROJECT_NAME}_URL}" "${${PROJECT_NAME}_ARCHIVE}.part"
    ${_download_args}
    STATUS _download_status
  )
  list(GET _download_status 0 _download_code)
  if(NOT _download_code EQUAL 0)
    file(REMOVE "${${PROJECT_NAME}_ARCHIVE}.part")
    list(GET _download_status 1 _download_error)
    message(FATAL_ERROR "Failed to download ${${PROJECT_NAME}_ARCHIVE_NAME}: ${_download_error}")
  endif()
  file(RENAME "${${PROJECT_NAME}_ARCHIVE}.part" "${${PROJECT_NAME}_ARCHIVE}")
endif()

set(${PROJECT_NAME}_EP_NAME "EP_${PROJECT_NAME}")
set(${PROJECT_NAME}_ARCHIVE_DIR
  "${CMAKE_CURRENT_BINARY_DIR}/${${PROJECT_NAME}_EP_NAME}-prefix/src/${${PROJECT_NAME}_EP_NAME}")

include(ExternalProject)
externalproject_add(${${PROJECT_NAME}_EP_NAME}
  URL "${${PROJECT_NAME}_ARCHIVE}"
  DOWNLOAD_EXTRACT_TIMESTAMP TRUE
  CONFIGURE_COMMAND ""
  BUILD_COMMAND ""
  UPDATE_COMMAND ""
  INSTALL_COMMAND ""
)

# Headers go in a package-specific directory, so they don't end up directly in a merged include directory
install(DIRECTORY "${${PROJECT_NAME}_ARCHIVE_DIR}/include/" DESTINATION include/${PROJECT_NAME})
# Preserve the archive's SONAME symlinks and install all runtime libraries, including the
# execution providers (shared, CUDA, TensorRT) that libonnxruntime loads at runtime.
# The archive's CMake config and pkg-config files are skipped, they point to the wrong directories.
install(DIRECTORY "${${PROJECT_NAME}_ARCHIVE_DIR}/lib/" DESTINATION lib
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
