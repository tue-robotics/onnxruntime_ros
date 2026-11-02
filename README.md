# onnxruntime_ros

This CMake project downloads a binary ONNX Runtime archive and installs its headers and libraries into the ROS 2 install prefix.

The package can be used in a ROS 2 workspace by adding:
```XML
  <depend>onnxruntime_ros</depend>
```
to your `package.xml`. Headers are installed to `include/onnxruntime_ros` and libraries to `lib` of the workspace install folder.

Usage in a CMake project:
```CMake
find_package(onnxruntime_ros REQUIRED)
target_link_libraries(${PROJECT_NAME} PUBLIC onnxruntime_ros::onnxruntime_ros)
```
`onnxruntime::onnxruntime` is also available, the same target name as the upstream ONNX Runtime CMake config and `onnxruntime_vendor`.
`onnxruntime_ros_CUDA_ENABLED` tells whether the GPU (CUDA/TensorRT) build of ONNX Runtime was installed.

If you get the error `No CMAKE_CUDA_COMPILER could be found.`, then the CUDA compiler `nvcc` cannot be found in the default search paths (`$PATH`). In this case, you have to set the path to `nvcc` manually:
```sh
export CUDACXX=/usr/local/cuda/bin/nvcc
```
