cmake_minimum_required(VERSION 3.24 FATAL_ERROR)
project(onnxruntime_ros)

cmake_policy(SET CMP0097 NEW)
cmake_policy(SET CMP0148 OLD)

find_package(catkin REQUIRED)

catkin_package_xml()
# split version in parts and pass to extra file
string(REPLACE "." ";" ${PROJECT_NAME}_VERSION_LIST "${${PROJECT_NAME}_VERSION}")
list(LENGTH ${PROJECT_NAME}_VERSION_LIST _count)
if(NOT _count EQUAL 3)
  message(FATAL_ERROR "${PROJECT_NAME} version '${${PROJECT_NAME}_VERSION}' does not match 'MAJOR.MINOR.PATCH' pattern")
endif()
list(GET ${PROJECT_NAME}_VERSION_LIST 0 ${PROJECT_NAME}_VERSION_MAJOR)
list(GET ${PROJECT_NAME}_VERSION_LIST 1 ${PROJECT_NAME}_VERSION_MINOR)
list(GET ${PROJECT_NAME}_VERSION_LIST 2 ${PROJECT_NAME}_VERSION_PATCH)

if(NOT ${CMAKE_VERBOSE_MAKEFILE})
  set(should_log ON)
else()
  set(should_log OFF)
endif()

set(ONNXRUNTIME_URL_BASE "https://github.com/microsoft/onnxruntime/releases/download/v${${PROJECT_NAME}_VERSION}")

set(EP_NAME "EP_${PROJECT_NAME}")
set(ONNXRUNTIME_SOURCE_DIR "${CMAKE_CURRENT_BINARY_DIR}/${EP_NAME}-prefix/src/${EP_NAME}")
set(ONNXRUNTIME_INSTALL_DIR "${CMAKE_CURRENT_BINARY_DIR}/${EP_NAME}_install")
set(ONNXRUNTIME_INCLUDE_DIR
    "${ONNXRUNTIME_INSTALL_DIR}/include"
    CACHE PATH "onnxruntime include directory." FORCE)
set(ONNXRUNTIME_LIB_DIR
    "${ONNXRUNTIME_INSTALL_DIR}/lib"
    CACHE PATH "onnxruntime lib directory." FORCE)
set(ONNXRUNTIME_LIB
    "${ONNXRUNTIME_LIB_DIR}/libonnxruntime.so"
    CACHE FILEPATH "ONNXRUNTIME shared library." FORCE)

set(_install_commands
  COMMAND ${CMAKE_COMMAND} -E copy_directory_if_different ${ONNXRUNTIME_SOURCE_DIR}/lib/cmake ${CATKIN_DEVEL_PREFIX}/${CATKIN_PACKAGE_SHARE_DESTINATION}/cmake
  COMMAND ${CMAKE_COMMAND} -E copy_directory_if_different ${ONNXRUNTIME_SOURCE_DIR}/include ${CATKIN_DEVEL_PREFIX}/${CATKIN_GLOBAL_INCLUDE_DESTINATION}
  COMMAND ${CMAKE_COMMAND} -E copy_if_different "${ONNXRUNTIME_SOURCE_DIR}/lib/libonnxruntime.so.${${PROJECT_NAME}_VERSION}" ${CATKIN_DEVEL_PREFIX}/${CATKIN_PACKAGE_LIB_DESTINATION}
  COMMAND ${CMAKE_COMMAND} -E create_symlink "${CATKIN_DEVEL_PREFIX}/${CATKIN_PACKAGE_LIB_DESTINATION}/libonnxruntime.so.${${PROJECT_NAME}_VERSION}" "${CATKIN_DEVEL_PREFIX}/${CATKIN_PACKAGE_LIB_DESTINATION}/libonnxruntime.so.${${PROJECT_NAME}_VERSION_MAJOR}"
  COMMAND ${CMAKE_COMMAND} -E create_symlink "${CATKIN_DEVEL_PREFIX}/${CATKIN_PACKAGE_LIB_DESTINATION}/libonnxruntime.so.${${PROJECT_NAME}_VERSION_MAJOR}" "${CATKIN_DEVEL_PREFIX}/${CATKIN_PACKAGE_LIB_DESTINATION}/libonnxruntime.so"
  COMMAND ${CMAKE_COMMAND} -E copy_if_different "${ONNXRUNTIME_SOURCE_DIR}/lib/libonnxruntime_providers_shared.so" ${CATKIN_DEVEL_PREFIX}/${CATKIN_PACKAGE_LIB_DESTINATION}
)
set(_build_byproducts
  "${CATKIN_DEVEL_PREFIX}/${CATKIN_PACKAGE_LIB_DESTINATION}/libonnxruntime.so.${${PROJECT_NAME}_VERSION}"
  "${CATKIN_DEVEL_PREFIX}/${CATKIN_PACKAGE_LIB_DESTINATION}/libonnxruntime.so.${${PROJECT_NAME}_VERSION_MAJOR}"
  "${CATKIN_DEVEL_PREFIX}/${CATKIN_PACKAGE_LIB_DESTINATION}/libonnxruntime.so"
  "${CATKIN_DEVEL_PREFIX}/${CATKIN_PACKAGE_LIB_DESTINATION}/libonnxruntime_providers_shared.so"
)
set(_install_files
  ${ONNXRUNTIME_LIB}
  ${ONNXRUNTIME_LIB}.${${PROJECT_NAME}_VERSION_MAJOR}
  ${ONNXRUNTIME_LIB}.${${PROJECT_NAME}_VERSION}
  "${ONNXRUNTIME_LIB_DIR}/libonnxruntime_providers_shared.so"
)


include(CheckLanguage)
check_language(CUDA)
if(CMAKE_CUDA_COMPILER)
  enable_language(CUDA)
  message(STATUS "Compiling with CUDA")
  set(ONNXRUNTIME_FILENAME "onnxruntime-linux-x64-gpu-${${PROJECT_NAME}_VERSION}.tgz")
  list(APPEND _install_commands
    COMMAND ${CMAKE_COMMAND} -E copy_if_different "${ONNXRUNTIME_SOURCE_DIR}/lib/libonnxruntime_providers_cuda.so" ${CATKIN_DEVEL_PREFIX}/${CATKIN_PACKAGE_LIB_DESTINATION}
    COMMAND ${CMAKE_COMMAND} -E copy_if_different "${ONNXRUNTIME_SOURCE_DIR}/lib/libonnxruntime_providers_tensorrt.so" ${CATKIN_DEVEL_PREFIX}/${CATKIN_PACKAGE_LIB_DESTINATION}
  )
  list(APPEND _build_byproducts
    "${CATKIN_DEVEL_PREFIX}/${CATKIN_PACKAGE_LIB_DESTINATION}/libonnxruntime_providers_cuda.so"
    "${CATKIN_DEVEL_PREFIX}/${CATKIN_PACKAGE_LIB_DESTINATION}/libonnxruntime_providers_tensorrt.so"
  )
  list(SORT _build_byproducts)
  list(APPEND _install_files
    "${ONNXRUNTIME_LIB_DIR}/libonnxruntime_providers_cuda.so"
    "${ONNXRUNTIME_LIB_DIR}/libonnxruntime_providers_tensorrt.so"
  )
  list(SORT _install_files)
else()
  message(STATUS "Compiling without CUDA")
  set(ONNXRUNTIME_FILENAME "onnxruntime-linux-x64-${${PROJECT_NAME}_VERSION}.tgz")
endif()

# Download at configure time instead of via externalproject_add, so a GITHUB_TOKEN is only kept in memory.
# externalproject_add would write its HTTP_HEADER to the generated download script in the build directory.
set(ONNXRUNTIME_ARCHIVE "${CMAKE_CURRENT_BINARY_DIR}/${ONNXRUNTIME_FILENAME}")
if(NOT EXISTS "${ONNXRUNTIME_ARCHIVE}")
  # Authenticate the download when a GITHUB_TOKEN is available, to avoid rate limiting
  # The token only authenticates the download and doesn't affect the build result, so silence catkin_lint's ENV_VAR warning
  #catkin_lint: ignore env_var
  set(_download_args)
  if(NOT "$ENV{GITHUB_TOKEN}" STREQUAL "")
    list(APPEND _download_args HTTPHEADER "Authorization: Bearer $ENV{GITHUB_TOKEN}")
  endif()
  message(STATUS "Downloading ${ONNXRUNTIME_URL_BASE}/${ONNXRUNTIME_FILENAME}")
  file(DOWNLOAD "${ONNXRUNTIME_URL_BASE}/${ONNXRUNTIME_FILENAME}" "${ONNXRUNTIME_ARCHIVE}.part"
    ${_download_args}
    STATUS _download_status
  )
  #catkin_lint: report env_var
  list(GET _download_status 0 _download_code)
  if(NOT _download_code EQUAL 0)
    file(REMOVE "${ONNXRUNTIME_ARCHIVE}.part")
    list(GET _download_status 1 _download_error)
    message(FATAL_ERROR "Failed to download ${ONNXRUNTIME_FILENAME}: ${_download_error}")
  endif()
  file(RENAME "${ONNXRUNTIME_ARCHIVE}.part" "${ONNXRUNTIME_ARCHIVE}")
endif()

include(ExternalProject)
externalproject_add(${EP_NAME}
  URL "${ONNXRUNTIME_ARCHIVE}"
  LOG_CONFIGURE ${should_log}
  LOG_BUILD ${should_log}
  CONFIGURE_COMMAND ""
  BUILD_COMMAND ""
  UPDATE_COMMAND ""
  INSTALL_COMMAND ${_install_commands}
  BUILD_BYPRODUCTS ${_build_byproducts}
)

install(
  DIRECTORY ${ONNXRUNTIME_LIB_DIR}/cmake/
  DESTINATION ${CATKIN_PACKAGE_SHARE_DESTINATION}/cmake
)
install(
  FILES "${ONNXRUNTIME_LIB_DIR}/pkgconfig/*.pc"
  DESTINATION ${CATKIN_GLOBAL_LIB_DESTINATION}/pkgconfig
)
install(
  DIRECTORY ${ONNXRUNTIME_SOURCE_DIR}/include/
  DESTINATION ${CATKIN_GLOBAL_INCLUDE_DESTINATION}
)
install(
  FILES ${_install_files}
  DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
)

if(CMAKE_CUDA_COMPILER)
    set(${PROJECT_NAME}_CUDA_ENABLED TRUE)
else()
    set(${PROJECT_NAME}_CUDA_ENABLED FALSE)
endif()

catkin_package(
  CFG_EXTRAS ${PROJECT_NAME}-extras.cmake
  # INCLUDE_DIRECTORIES SYSTEM include
  EXPORTED_TARGETS ${EP_NAME}
)

if(CATKIN_ENABLE_TESTING)
  find_package(catkin_lint_cmake REQUIRED)
  catkin_add_catkin_lint_test("-W2 --ignore GLOBAL_VAR_COLLISION --ignore MISSING_DIRECTORY --ignore MISSING_FILE --ignore UNDEFINED_TARGET")
endif()
