cmake_minimum_required(VERSION 3.24 FATAL_ERROR)
project(onnxruntime_ros)

find_package(ament_cmake REQUIRED)
include(GNUInstallDirs)
ament_package_xml()
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
  COMMAND ${CMAKE_COMMAND} -E make_directory ${ONNXRUNTIME_INSTALL_DIR}/lib
  COMMAND ${CMAKE_COMMAND} -E make_directory ${ONNXRUNTIME_INSTALL_DIR}/${CMAKE_INSTALL_LIBDIR}
  COMMAND ${CMAKE_COMMAND} -E copy_directory_if_different
    ${ONNXRUNTIME_SOURCE_DIR}/lib/cmake ${ONNXRUNTIME_INSTALL_DIR}/lib/cmake
  COMMAND ${CMAKE_COMMAND} -E copy_directory_if_different
    ${ONNXRUNTIME_SOURCE_DIR}/include ${ONNXRUNTIME_INSTALL_DIR}/include
  COMMAND ${CMAKE_COMMAND} -E copy_if_different
    "${ONNXRUNTIME_SOURCE_DIR}/lib/libonnxruntime.so.${${PROJECT_NAME}_VERSION}"
    ${ONNXRUNTIME_INSTALL_DIR}/${CMAKE_INSTALL_LIBDIR}
  COMMAND ${CMAKE_COMMAND} -E create_symlink
    "libonnxruntime.so.${${PROJECT_NAME}_VERSION}"
    ${ONNXRUNTIME_INSTALL_DIR}/${CMAKE_INSTALL_LIBDIR}/libonnxruntime.so.${${PROJECT_NAME}_VERSION_MAJOR}
  COMMAND ${CMAKE_COMMAND} -E create_symlink
    "libonnxruntime.so.${${PROJECT_NAME}_VERSION_MAJOR}"
    ${ONNXRUNTIME_INSTALL_DIR}/${CMAKE_INSTALL_LIBDIR}/libonnxruntime.so
  COMMAND ${CMAKE_COMMAND} -E copy_if_different
    "${ONNXRUNTIME_SOURCE_DIR}/lib/libonnxruntime_providers_shared.so"
    ${ONNXRUNTIME_INSTALL_DIR}/${CMAKE_INSTALL_LIBDIR}
)
set(_build_byproducts
  "${ONNXRUNTIME_INSTALL_DIR}/${CMAKE_INSTALL_LIBDIR}/libonnxruntime.so.${${PROJECT_NAME}_VERSION}"
  "${ONNXRUNTIME_INSTALL_DIR}/${CMAKE_INSTALL_LIBDIR}/libonnxruntime.so.${${PROJECT_NAME}_VERSION_MAJOR}"
  "${ONNXRUNTIME_INSTALL_DIR}/${CMAKE_INSTALL_LIBDIR}/libonnxruntime.so"
  "${ONNXRUNTIME_INSTALL_DIR}/${CMAKE_INSTALL_LIBDIR}/libonnxruntime_providers_shared.so"
)
set(_install_files
  "${ONNXRUNTIME_INSTALL_DIR}/${CMAKE_INSTALL_LIBDIR}/libonnxruntime.so"
  "${ONNXRUNTIME_INSTALL_DIR}/${CMAKE_INSTALL_LIBDIR}/libonnxruntime.so.${${PROJECT_NAME}_VERSION_MAJOR}"
  "${ONNXRUNTIME_INSTALL_DIR}/${CMAKE_INSTALL_LIBDIR}/libonnxruntime.so.${${PROJECT_NAME}_VERSION}"
  "${ONNXRUNTIME_INSTALL_DIR}/${CMAKE_INSTALL_LIBDIR}/libonnxruntime_providers_shared.so"
)


include(CheckLanguage)
check_language(CUDA)
if(CMAKE_CUDA_COMPILER)
  enable_language(CUDA)
  message(STATUS "Compiling with CUDA")
  set(ONNXRUNTIME_FILENAME "onnxruntime-linux-x64-gpu-${${PROJECT_NAME}_VERSION}.tgz")
    list(APPEND _install_commands
    COMMAND ${CMAKE_COMMAND} -E copy_if_different
      "${ONNXRUNTIME_SOURCE_DIR}/lib/libonnxruntime_providers_cuda.so"
      ${ONNXRUNTIME_INSTALL_DIR}/${CMAKE_INSTALL_LIBDIR}
    COMMAND ${CMAKE_COMMAND} -E copy_if_different
      "${ONNXRUNTIME_SOURCE_DIR}/lib/libonnxruntime_providers_tensorrt.so"
      ${ONNXRUNTIME_INSTALL_DIR}/${CMAKE_INSTALL_LIBDIR}
  )
  list(APPEND _build_byproducts
    "${ONNXRUNTIME_INSTALL_DIR}/${CMAKE_INSTALL_LIBDIR}/libonnxruntime_providers_cuda.so"
    "${ONNXRUNTIME_INSTALL_DIR}/${CMAKE_INSTALL_LIBDIR}/libonnxruntime_providers_tensorrt.so"
  )
  list(SORT _build_byproducts)
  list(APPEND _install_files
    "${ONNXRUNTIME_INSTALL_DIR}/${CMAKE_INSTALL_LIBDIR}/libonnxruntime_providers_cuda.so"
    "${ONNXRUNTIME_INSTALL_DIR}/${CMAKE_INSTALL_LIBDIR}/libonnxruntime_providers_tensorrt.so"
  )
  list(SORT _install_files)
else()
  message(STATUS "Compiling without CUDA")
  set(ONNXRUNTIME_FILENAME "onnxruntime-linux-x64-${${PROJECT_NAME}_VERSION}.tgz")
endif()

include(ExternalProject)
externalproject_add(${EP_NAME}
  URL "${ONNXRUNTIME_URL_BASE}/${ONNXRUNTIME_FILENAME}"
  DOWNLOAD_NO_PROGRESS NOT ${should_log}
  LOG_CONFIGURE ${should_log}
  LOG_BUILD ${should_log}
  CONFIGURE_COMMAND ""
  BUILD_COMMAND ""
  UPDATE_COMMAND ""
  INSTALL_COMMAND ${_install_commands}
  BUILD_BYPRODUCTS ${_build_byproducts}
)

add_custom_target(${PROJECT_NAME}_external ALL DEPENDS ${EP_NAME})

install(
  DIRECTORY ${ONNXRUNTIME_INSTALL_DIR}/lib/cmake/
  DESTINATION ${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME}/cmake
)
install(
  DIRECTORY ${ONNXRUNTIME_SOURCE_DIR}/lib/pkgconfig/
  DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig
  FILES_MATCHING PATTERN "*.pc"
)
install(
  DIRECTORY ${ONNXRUNTIME_INSTALL_DIR}/include/
  DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
)
install(
  FILES ${_install_files}
  DESTINATION ${CMAKE_INSTALL_LIBDIR}
)

if(CMAKE_CUDA_COMPILER)
    set(${PROJECT_NAME}_CUDA_ENABLED TRUE)
else()
    set(${PROJECT_NAME}_CUDA_ENABLED FALSE)
endif()

configure_file(
  cmake/${PROJECT_NAME}-extras.cmake.in
  ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}-extras.cmake
  @ONLY
)

if(BUILD_TESTING)
  find_package(ament_lint_auto REQUIRED)
  ament_lint_auto_find_test_dependencies()
endif()

ament_package(CONFIG_EXTRAS "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}-extras.cmake")
