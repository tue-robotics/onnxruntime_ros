cmake_minimum_required(VERSION 3.24 FATAL_ERROR)
project(onnxruntime_ros CXX)

cmake_policy(SET CMP0097 NEW)
#cmake_policy(SET CMP0148 OLD)

find_package(catkin REQUIRED
  COMPONENTS
)

catkin_package_xml()
# split version in parts and pass to extra file
string(REPLACE "." ";" onnxruntime_ros_VERSION_LIST "${onnxruntime_ros_VERSION}")
list(LENGTH onnxruntime_ros_VERSION_LIST _count)
if(NOT _count EQUAL 3)
  message(FATAL_ERROR "onnxruntime_ros version '${onnxruntime_ros_VERSION}' does not match 'MAJOR.MINOR.PATCH' pattern")
endif()
list(GET onnxruntime_ros_VERSION_LIST 0 onnxruntime_ros_VERSION_MAJOR)
list(GET onnxruntime_ros_VERSION_LIST 1 onnxruntime_ros_VERSION_MINOR)
list(GET onnxruntime_ros_VERSION_LIST 2 onnxruntime_ros_VERSION_PATCH)

macro(build_onnxruntime)
  # set(extra_cmake_args)

  # if(DEFINED CMAKE_BUILD_TYPE)
    # list(APPEND extra_cmake_args -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE})
  # endif()

  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -w")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -w")

  # list(APPEND extra_cmake_args "-Donnxruntime_BUILD_UNIT_TESTS=OFF")
  # # list(APPEND extra_cmake_args "-Donnxruntime_BUILD_SHARED_LIB=ON")
  # list(APPEND extra_cmake_args "-DCMAKE_CXX_COMPILER=${CMAKE_CXX_COMPILER}")
  # list(APPEND extra_cmake_args "-DCMAKE_CXX_FLAGS=${CMAKE_CXX_FLAGS}")

  if(NOT ${CMAKE_VERBOSE_MAKEFILE})
    set(should_log ON)
  else()
    set(should_log OFF)
  endif()

  # if(DEFINED CMAKE_TOOLCHAIN_FILE)
    # list(APPEND extra_cmake_args "-DCMAKE_TOOLCHAIN_FILE=${CMAKE_TOOLCHAIN_FILE}")
  # endif()

  set(ONNXRUNTIME_URL_BASE "https://github.com/microsoft/onnxruntime/releases/download/v${onnxruntime_ros_VERSION}")

  include(CheckLanguage)
  check_language(CUDA)
  if(CMAKE_CUDA_COMPILER)
    message(STATUS "Compiling with CUDA")
    set(ONNXRUNTIME_FILENAME "onnxruntime-linux-x64-gpu-${onnxruntime_ros_VERSION}.tgz")
    # Add additional cuda related cmake variables
    # list(APPEND extra_cmake_args "-Donnxruntime_USE_CUDA=ON")
    # list(APPEND extra_cmake_args "-Donnxruntime_CUDA_HOME=/usr/local/cuda/")
    # list(APPEND extra_cmake_args "-DCUDA_CUDA_LIBRARY=/usr/local/cuda/lib64/stubs/")
    # list(APPEND extra_cmake_args "-Donnxruntime_USE_FULL_PROTOBUF=ON")
  else()
    message(STATUS "Compiling without CUDA")
    set(ONNXRUNTIME_FILENAME "onnxruntime-linux-x64-${onnxruntime_ros_VERSION}.tgz")
  endif()

  SET(EP_NAME "onnxruntime")

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
  ExternalProject_Add(${EP_NAME}
    URL ${ONNXRUNTIME_URL_BASE}/${ONNXRUNTIME_FILENAME}
    URL_MD5 9e29988a3148f31a4379248d0a1b68d9
    DOWNLOAD_NO_PROGRESS NOT ${should_log}
    LOG_CONFIGURE ${should_log}
    LOG_BUILD ${should_log}
    CONFIGURE_COMMAND ""
    BUILD_COMMAND ""
    UPDATE_COMMAND ""
    INSTALL_COMMAND
      # ${CMAKE_COMMAND} -E remove_directory ${ONNXRUNTIME_INSTALL_DIR} &&
      ${CMAKE_COMMAND} -E make_directory ${ONNXRUNTIME_INSTALL_DIR} &&
      ${CMAKE_COMMAND} -E create_symlink ${ONNXRUNTIME_SOURCE_DIR}/lib/ ${ONNXRUNTIME_LIB_DIR} &&
      ${CMAKE_COMMAND} -E create_symlink ${ONNXRUNTIME_SOURCE_DIR}/include ${ONNXRUNTIME_INCLUDE_DIR}
    BUILD_BYPRODUCTS ${ONNXRUNTIME_LIB}
  )

  # The external project will install to the build folder, but we'll install that on make install.
  # install(
  #   DIRECTORY
  #     ${ONNXRUNTIME_INSTALL_DIR}
  #   DESTINATION
  #     ${CMAKE_INSTALL_PREFIX}/opt/onnxruntime_ros
  #   USE_SOURCE_PERMISSIONS
  # )
  install(
    DIRECTORY ${ONNXRUNTIME_LIB_DIR}/cmake/
    DESTINATION ${CATKIN_PACKAGE_SHARE_DESTINATION}/cmake
  )
  install(
    FILES "${ONNXRUNTIME_LIB_DIR}/pkgconfig/*.pc"
    DESTINATION ${CATKIN_GLOBAL_LIB_DESTINATION}/pkgconfig
  )
  install(
    DIRECTORY ${ONNXRUNTIME_INCLUDE_DIR}
    DESTINATION ${CATKIN_PACKAGE_INCLUDE_DESTINATION}
  )
  install(
    FILES ${ONNXRUNTIME_LIB}
    DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
  )
endmacro()

build_onnxruntime()

#find_package(catkin REQUIRED
  #COMPONENTS
#)

#catkin_package_xml()
## split version in parts and pass to extra file
#string(REPLACE "." ";" onnxruntime_ros_VERSION_LIST "${onnxruntime_ros_VERSION}")
#list(LENGTH onnxruntime_ros_VERSION_LIST _count)
#if(NOT _count EQUAL 3)
  #message(FATAL_ERROR "onnxruntime_ros version '${onnxruntime_ros_VERSION}' does not match 'MAJOR.MINOR.PATCH' pattern")
#endif()
#list(GET onnxruntime_ros_VERSION_LIST 0 onnxruntime_ros_VERSION_MAJOR)
#list(GET onnxruntime_ros_VERSION_LIST 1 onnxruntime_ros_VERSION_MINOR)
#list(GET onnxruntime_ros_VERSION_LIST 2 onnxruntime_ros_VERSION_PATCH)

catkin_add_env_hooks(99.onnxruntime_ros_library_path SHELLS bash DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/env-hooks)
if(APPLE)
  set(ENV_VAR_NAME "DYLD_LIBRARY_PATH")
else()
  set(ENV_VAR_NAME "LD_LIBRARY_PATH")
endif()
set(ENV_VAR_VALUE "opt/onnxruntime_ros/lib")
catkin_add_env_hooks(99.onnxruntime_ros_library_path.dsv.in DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/env-hooks)

catkin_package(
  CFG_EXTRAS "onnxruntime_ros-extras.cmake.in"
  # INCLUDE_DIRECTORIES SYSTEM include
  LIBRARIES onnxruntime
)

if(CATKIN_ENABLE_TESTING)
  find_package(catkin_lint_cmake REQUIRED)
  catkin_add_catkin_lint_test("-W2")
endif()
