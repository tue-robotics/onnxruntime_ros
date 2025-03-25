cmake_minimum_required(VERSION 3.24 FATAL_ERROR)
project(torch_cpp CXX)

include(CheckLanguage)

check_language(CUDA)
if(CMAKE_CUDA_COMPILER)
    message(STATUS "Compiling with CUDA")
    set(TORCH_URL https://download.pytorch.org/libtorch/cu126/libtorch-cxx11-abi-shared-with-deps-2.6.0%2Bcu126.zip)
    set(TORCH_URL_MD5 5a63c3288d750b5bc47232dbcdd2bbab)
else(CUDA_FOUND)
    message(STATUS "Compiling without CUDA")
    set(TORCH_URL https://download.pytorch.org/libtorch/cpu/libtorch-cxx11-abi-shared-with-deps-2.6.0%2Bcpu.zip)
    set(TORCH_URL_MD5 a614e60fda44af1a96b193d3f53944f9)
endif()

include(ExternalProject)
ExternalProject_Add(torch_archive
  URL               ${TORCH_URL}
  URL_MD5           ${TORCH_URL_MD5}
  PREFIX            torch_archive
  CONFIGURE_COMMAND ""
  BUILD_COMMAND     ""
  INSTALL_COMMAND   ""
  DOWNLOAD_EXTRACT_TIMESTAMP TRUE
)

ExternalProject_Get_Property(torch_archive SOURCE_DIR)

set(TORCH_PATH ${SOURCE_DIR})

# manually install header, libraries and config files
install(DIRECTORY ${TORCH_PATH}/include/
    DESTINATION include
    FILES_MATCHING PATTERN "*.h*"
)

install(DIRECTORY ${TORCH_PATH}/lib/
    DESTINATION lib
)

install(DIRECTORY ${TORCH_PATH}/share/
    DESTINATION share
)
