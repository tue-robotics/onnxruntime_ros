# onnxruntime_ros

This CMake project downloads a binary ONNX Runtime archive and installs its headers, libraries, and CMake configuration files into the ROS 2 install prefix.

The package can be used in a ROS 2 colcon workspace by adding:
```XML
  <depend>onnxruntime_ros</depend>
  <export>
    <build_type>ament_cmake</build_type>
  </export>
```
in your `package.xml`. The relevant files will then be installed within the workspace's `install` folder.

Usage in a CMake project:
```CMake
find_package(onnxruntime_ros REQUIRED)
target_include_directories(${PROJECT_NAME} PUBLIC ${onnxruntime_ros_INCLUDE_DIRS})
target_link_libraries(${PROJECT_NAME} PUBLIC ${onnxruntime_ros_LIBRARIES})
```

If you get the error `No CMAKE_CUDA_COMPILER could be found.`, then the CUDA compiler `nvcc` cannot be found in the default search paths (`$PATH`). In this case, you have to set the path to `nvcc` manually:
```sh
export CUDACXX=/usr/local/cuda/bin/nvcc
```
