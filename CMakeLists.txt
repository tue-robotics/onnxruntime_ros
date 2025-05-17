cmake_minimum_required(VERSION 3.16 FATAL_ERROR)
project(onnxruntime_ros CXX)

cmake_policy(SET CMP0097 NEW)

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
  set(extra_cmake_args)

  if(DEFINED CMAKE_BUILD_TYPE)
    list(APPEND extra_cmake_args -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE})
  endif()

  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -w")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -w")

  # list(APPEND extra_cmake_args "-Donnxruntime_BUILD_UNIT_TESTS=OFF")
  list(APPEND extra_cmake_args "-Donnxruntime_BUILD_SHARED_LIB=ON")
  list(APPEND extra_cmake_args "-DCMAKE_CXX_COMPILER=${CMAKE_CXX_COMPILER}")
  list(APPEND extra_cmake_args "-DCMAKE_CXX_FLAGS=${CMAKE_CXX_FLAGS}")

  # TODO(anyone): Verify Windows build
  if(WIN32 AND NOT ${CMAKE_VERBOSE_MAKEFILE})
    set(should_log ON)  # prevent warnings in Windows CI
  else()
    set(should_log OFF)
  endif()

  if(DEFINED CMAKE_TOOLCHAIN_FILE)
    list(APPEND extra_cmake_args "-DCMAKE_TOOLCHAIN_FILE=${CMAKE_TOOLCHAIN_FILE}")
  endif()

  include(CheckLanguage)
  check_language(CUDA)
  if(CMAKE_CUDA_COMPILER)
    message(STATUS "Compiling with CUDA")
    # Add additional cuda related cmake variables
    list(APPEND extra_cmake_args "-Donnxruntime_USE_CUDA=ON")
    list(APPEND extra_cmake_args "-Donnxruntime_CUDA_HOME=/usr/local/cuda/")
    list(APPEND extra_cmake_args "-DCUDA_CUDA_LIBRARY=/usr/local/cuda/lib64/stubs/")
    list(APPEND extra_cmake_args "-Donnxruntime_CUDNN_HOME=/usr")
    list(APPEND extra_cmake_args "-Donnxruntime_USE_FULL_PROTOBUF=ON")
  else()
    message(STATUS "Compiling without CUDA")
  endif()

  include(ExternalProject)
  ExternalProject_Add(onnxruntime
    GIT_REPOSITORY https://github.com/microsoft/onnxruntime
    GIT_TAG v1.22.0
    GIT_SUBMODULES ""
    GIT_SUBMODULES_RECURSE OFF
    LOG_CONFIGURE ${should_log}
    LOG_BUILD ${should_log}
    SOURCE_SUBDIR cmake
    CMAKE_ARGS
      -DCMAKE_INSTALL_PREFIX=${CMAKE_CURRENT_BINARY_DIR}/onnxruntime_install
      ${extra_cmake_args}
      -Wno-dev
  )

  # The external project will install to the build folder, but we'll install that on make install.
  install(
    DIRECTORY
      ${CMAKE_CURRENT_BINARY_DIR}/onnxruntime_install/
    DESTINATION
      ${CMAKE_INSTALL_PREFIX}/opt/onnxruntime_ros
    USE_SOURCE_PERMISSIONS
  )
endmacro()

build_onnxruntime()

# TODO(Briancbn): environment variables in Windows
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
)
