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

macro(build_onnxruntime)
  if(NOT ${CMAKE_VERBOSE_MAKEFILE})
    set(should_log ON)
  else()
    set(should_log OFF)
  endif()

  set(ONNXRUNTIME_URL_BASE "https://github.com/microsoft/onnxruntime/releases/download/v${${PROJECT_NAME}_VERSION}")

  include(CheckLanguage)
  check_language(CUDA)
  if(CMAKE_CUDA_COMPILER)
    enable_language(CUDA)
    message(STATUS "Compiling with CUDA")
    set(ONNXRUNTIME_FILENAME "onnxruntime-linux-x64-gpu-${${PROJECT_NAME}_VERSION}.tgz")
  else()
    message(STATUS "Compiling without CUDA")
    set(ONNXRUNTIME_FILENAME "onnxruntime-linux-x64-${${PROJECT_NAME}_VERSION}.tgz")
  endif()

  set(EP_NAME "EP_${PROJECT_NAME}")

  set(ONNXRUNTIME_SOURCE_DIR "${CMAKE_CURRENT_BINARY_DIR}/${EP_NAME}-prefix/src/${EP_NAME}")
  set(ONNXRUNTIME_INSTALL_DIR "${CMAKE_CURRENT_BINARY_DIR}/${EP_NAME}_install")

  set(ONNXRUNTIME_INCLUDE_DIR
      "${ONNXRUNTIME_INSTALL_DIR}/include"
      CACHE PATH "onnxruntime include directory." FORCE)
  set(ONNXRUNTIME_LIB_DIR
      "${ONNXRUNTIME_INSTALL_DIR}/lib"
      CACHE PATH "onnxruntime lib directory." FORCE)

  if(WIN32)
    set(ONNXRUNTIME_LIB
        "${ONNXRUNTIME_LIB_DIR}/onnxruntime.lib"
        CACHE FILEPATH "ONNXRUNTIME shared library." FORCE)
  elseif(APPLE)
    set(ONNXRUNTIME_LIB
        "${ONNXRUNTIME_LIB_DIR}/libonnxruntime.dylib"
        CACHE FILEPATH "ONNXRUNTIME shared library." FORCE)
  else()
    set(ONNXRUNTIME_LIB
        "${ONNXRUNTIME_LIB_DIR}/libonnxruntime.so"
        CACHE FILEPATH "ONNXRUNTIME shared library." FORCE)
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
    INSTALL_COMMAND
      # COMMAND ${CMAKE_COMMAND} -E remove_directory ${ONNXRUNTIME_INSTALL_DIR}
      # COMMAND ${CMAKE_COMMAND} -E make_directory ${ONNXRUNTIME_INSTALL_DIR}
      COMMAND ${CMAKE_COMMAND} -E copy_directory_if_different ${ONNXRUNTIME_SOURCE_DIR}/lib/cmake ${CATKIN_DEVEL_PREFIX}/${CATKIN_PACKAGE_SHARE_DESTINATION}/cmake
      COMMAND ${CMAKE_COMMAND} -E copy_directory_if_different ${ONNXRUNTIME_SOURCE_DIR}/include ${CATKIN_DEVEL_PREFIX}/${CATKIN_GLOBAL_INCLUDE_DESTINATION}
      COMMAND ${CMAKE_COMMAND} -E copy_if_different "${ONNXRUNTIME_SOURCE_DIR}/lib/libonnxruntime.so" ${CATKIN_DEVEL_PREFIX}/${CATKIN_PACKAGE_LIB_DESTINATION}
    BUILD_BYPRODUCTS ${ONNXRUNTIME_LIB}
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
    FILES ${ONNXRUNTIME_LIB}
    DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
  )
endmacro()

build_onnxruntime()

# catkin_add_env_hooks(99.onnxruntime_ros_library_path SHELLS bash DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/env-hooks)
# if(APPLE)
#   set(ENV_VAR_NAME "DYLD_LIBRARY_PATH")
# else()
#   set(ENV_VAR_NAME "LD_LIBRARY_PATH")
# endif()
# set(ENV_VAR_VALUE "opt/onnxruntime_ros/lib")
# catkin_add_env_hooks(99.onnxruntime_ros_library_path.dsv.in DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/env-hooks)

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
  catkin_add_catkin_lint_test("-W2")
endif()
