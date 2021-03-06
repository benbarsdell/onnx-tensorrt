 # Copyright (c) 2018, NVIDIA CORPORATION. All rights reserved.
 #
 # Permission is hereby granted, free of charge, to any person obtaining a
 # copy of this software and associated documentation files (the "Software"),
 # to deal in the Software without restriction, including without limitation
 # the rights to use, copy, modify, merge, publish, distribute, sublicense,
 # and/or sell copies of the Software, and to permit persons to whom the
 # Software is furnished to do so, subject to the following conditions:
 #
 # The above copyright notice and this permission notice shall be included in
 # all copies or substantial portions of the Software.
 #
 # THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
 # IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
 # FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT.  IN NO EVENT SHALL
 # THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
 # LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING
 # FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER
 # DEALINGS IN THE SOFTWARE.

cmake_minimum_required(VERSION 3.2 FATAL_ERROR)
project(onnx2trt LANGUAGES CXX C)
set(ONNX2TRT_ROOT ${PROJECT_SOURCE_DIR})
# Set C++11 as standard for the whole project
set(CMAKE_CXX_STANDARD  11)

# Build the libraries with -fPIC
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

set(PARSER_LINKER_SCRIPT  ${ONNX2TRT_ROOT}/libnvonnxparser.version)
set(RUNTIME_LINKER_SCRIPT ${ONNX2TRT_ROOT}/libnvonnxparser_runtime.version)

#--------------------------------------------------
# Version information
#--------------------------------------------------
set(ONNX2TRT_MAJOR 0)
set(ONNX2TRT_MINOR 1)
set(ONNX2TRT_PATCH 0)

#--------------------------------------------------
# Build configurations, global to all projects
#--------------------------------------------------

set(PLUGIN_SOURCES
  FancyActivation.cu
  ResizeNearest.cu
  Split.cu
  InstanceNormalization.cpp
  plugin.cpp
  )

set(IMPORTER_SOURCES
  NvOnnxParser.cpp
  ModelImporter.cpp
  builtin_op_importers.cpp
  onnx2trt_utils.cpp
  ShapedWeights.cpp
  OnnxAttrs.cpp
)
set(RUNTIME_SOURCES
  NvOnnxParserRuntime.cpp
  PluginFactory.cpp
  builtin_plugins.cpp
)
set(EXECUTABLE_SOURCES
  main.cpp
)

set(HEADERS
  NvOnnxParser.h
  NvOnnxParserRuntime.h
)

FIND_PACKAGE(Protobuf REQUIRED)

if(NOT TARGET onnx_proto)
  # Note: This avoids libprotobuf.so complaining about name collisions at runtime
  if(NOT ONNX_NAMESPACE)
    set(ONNX_NAMESPACE "onnx2trt_onnx")
  endif()
  add_definitions("-DONNX_NAMESPACE=${ONNX_NAMESPACE}")
  add_subdirectory(third_party/onnx EXCLUDE_FROM_ALL)
endif()

#
# CUDA Configuration
#
find_package(CUDA REQUIRED)
list(APPEND GPU_ARCHS
    35
    53
    61
    70
    )

set(CUDA_VERBOSE_BUILD ON)

# Generate SASS for each architecture
foreach(arch ${GPU_ARCHS})
    set(GENCODES "${GENCODES} -gencode arch=compute_${arch},code=sm_${arch}")
endforeach()
# Generate PTX for the last architecture
list(GET GPU_ARCHS -1 LATEST_GPU_ARCH)
set(GENCODES "${GENCODES} -gencode arch=compute_${LATEST_GPU_ARCH},code=compute_${LATEST_GPU_ARCH}")

set(CMAKE_CUDA_FLAGS "${CMAKE_CUDA_FLAGS} \
    -cudart static \
    -lineinfo \
    -g \
    --expt-extended-lambda \
    ${GENCODES} \
")

# Specify the cuda host compiler to use the same compiler as cmake.
set(CUDA_HOST_COMPILER ${CMAKE_CXX_COMPILER})


# CUDNN
set(CUDNN_ROOT_DIR "" CACHE PATH "Folder contains NVIDIA cuDNN")
find_path(CUDNN_INCLUDE_DIR cudnn.h
    HINTS ${CUDNN_ROOT_DIR} ${CUDA_TOOLKIT_ROOT_DIR}
    PATH_SUFFIXES cuda/include include)
find_library(CUDNN_LIBRARY cudnn
    HINTS ${CUDNN_ROOT_DIR} ${CUDA_TOOLKIT_ROOT_DIR}
    PATH_SUFFIXES lib lib64 cuda/lib cuda/lib64 lib/x64)
find_package_handle_standard_args(
    CUDNN DEFAULT_MSG CUDNN_INCLUDE_DIR CUDNN_LIBRARY)

if(NOT CUDNN_FOUND)
  message(WARNING
      "Cudnn cannot be found. TensorRT depends explicitly "
      "on cudnn so you should consider installing it.")
  return()
endif()

# TensorRT
find_path(TENSORRT_INCLUDE_DIR NvInfer.h
  HINTS ${TENSORRT_ROOT} ${CUDA_TOOLKIT_ROOT_DIR}
  PATH_SUFFIXES include)
MESSAGE(STATUS "Found TensorRT headers at ${TENSORRT_INCLUDE_DIR}")
find_library(TENSORRT_LIBRARY_INFER nvinfer
  HINTS ${TENSORRT_ROOT} ${TENSORRT_BUILD} ${CUDA_TOOLKIT_ROOT_DIR}
  PATH_SUFFIXES lib lib64 lib/x64)
find_library(TENSORRT_LIBRARY_INFER_PLUGIN nvinfer_plugin
  HINTS  ${TENSORRT_ROOT} ${TENSORRT_BUILD} ${CUDA_TOOLKIT_ROOT_DIR}
  PATH_SUFFIXES lib lib64 lib/x64)
set(TENSORRT_LIBRARY ${TENSORRT_LIBRARY_INFER} ${TENSORRT_LIBRARY_INFER_PLUGIN})
MESSAGE(STATUS "Find TensorRT libs at ${TENSORRT_LIBRARY}")
find_package_handle_standard_args(
  TENSORRT DEFAULT_MSG TENSORRT_INCLUDE_DIR TENSORRT_LIBRARY)
if(NOT TENSORRT_FOUND)
  message(ERROR
    "Cannot find TensorRT library.")
endif()


# --------------------------------
# Plugin library
# --------------------------------
if(NOT "${CUDA_NVCC_FLAGS}" MATCHES "-std=c\\+\\+11" )
  list(APPEND CUDA_NVCC_FLAGS -std=c++11)
endif()
list(APPEND CUDA_NVCC_FLAGS "-Xcompiler -fPIC --expt-extended-lambda")
CUDA_INCLUDE_DIRECTORIES(${CUDNN_INCLUDE_DIR} ${TENSORRT_INCLUDE_DIR})
CUDA_ADD_LIBRARY(nvonnxparser_plugin ${PLUGIN_SOURCES})
target_include_directories(nvonnxparser_plugin PUBLIC ${CUDA_INCLUDE_DIRS} ${ONNX_INCLUDE_DIRS} ${TENSORRT_INCLUDE_DIR} ${CUDNN_INCLUDE_DIR})
target_link_libraries(nvonnxparser_plugin ${TENSORRT_LIBRARY})

# --------------------------------
# Importer library
# --------------------------------
add_library(nvonnxparser SHARED ${IMPORTER_SOURCES})
target_include_directories(nvonnxparser PUBLIC ${CUDA_INCLUDE_DIRS} ${ONNX_INCLUDE_DIRS} ${TENSORRT_INCLUDE_DIR} ${CUDNN_INCLUDE_DIR})
target_link_libraries(nvonnxparser PUBLIC onnx_proto nvonnxparser_plugin ${PROTOBUF_LIBRARY} ${CUDNN_LIBRARY} ${TENSORRT_LIBRARY})
set_target_properties(nvonnxparser PROPERTIES
  VERSION   ${ONNX2TRT_MAJOR}.${ONNX2TRT_MINOR}.${ONNX2TRT_PATCH}
  SOVERSION ${ONNX2TRT_MAJOR}
  LINK_DEPENDS ${PARSER_LINKER_SCRIPT}
  LINK_FLAGS "-Wl,--version-script=${PARSER_LINKER_SCRIPT}"
)
add_library(nvonnxparser_static STATIC ${IMPORTER_SOURCES})
target_include_directories(nvonnxparser_static PUBLIC ${CUDA_INCLUDE_DIRS} ${ONNX_INCLUDE_DIRS} ${TENSORRT_INCLUDE_DIR} ${CUDNN_INCLUDE_DIR})
target_link_libraries(nvonnxparser_static PUBLIC onnx_proto nvonnxparser_plugin ${PROTOBUF_LIBRARY} ${CUDNN_LIBRARY} ${TENSORRT_LIBRARY})

# --------------------------------
# Runtime library
# --------------------------------
add_library(nvonnxparser_runtime SHARED ${RUNTIME_SOURCES})
target_include_directories(nvonnxparser_runtime PUBLIC ${CUDA_INCLUDE_DIRS} ${TENSORRT_INCLUDE_DIR} ${CUDNN_INCLUDE_DIR})
target_link_libraries(nvonnxparser_runtime PUBLIC nvonnxparser_plugin ${CUDNN_LIBRARY} ${TENSORRT_LIBRARY})
set_target_properties(nvonnxparser_runtime PROPERTIES
  VERSION   ${ONNX2TRT_MAJOR}.${ONNX2TRT_MINOR}.${ONNX2TRT_PATCH}
  SOVERSION ${ONNX2TRT_MAJOR}
  LINK_DEPENDS ${RUNTIME_LINKER_SCRIPT}
  LINK_FLAGS "-Wl,--version-script=${RUNTIME_LINKER_SCRIPT}"
)
add_library(nvonnxparser_runtime_static STATIC ${RUNTIME_SOURCES})
target_include_directories(nvonnxparser_runtime_static PUBLIC ${CUDA_INCLUDE_DIRS} ${TENSORRT_INCLUDE_DIR} ${CUDNN_INCLUDE_DIR})
target_link_libraries(nvonnxparser_runtime_static PUBLIC nvonnxparser_plugin ${CUDNN_LIBRARY} ${TENSORRT_LIBRARY})

# --------------------------------
# Converter executable
# --------------------------------

add_executable(onnx2trt ${EXECUTABLE_SOURCES})
target_include_directories(onnx2trt PUBLIC ${ONNX_INCLUDE_DIRS} ${CUDNN_INCLUDE_DIR})

target_link_libraries(onnx2trt PUBLIC nvonnxparser_static ${CUDA_LIBRARIES} -lrt -lpthread)

# --------------------------------
# Installation
# --------------------------------
install(TARGETS onnx2trt
                nvonnxparser        nvonnxparser_runtime
                nvonnxparser_static nvonnxparser_runtime_static
        RUNTIME DESTINATION bin
        LIBRARY DESTINATION lib
        ARCHIVE DESTINATION lib
)

install(FILES ${HEADERS}
  DESTINATION include
)

SET(CPACK_GENERATOR "DEB")
SET(CPACK_DEBIAN_PACKAGE_MAINTAINER "Mike Houston") #required
SET(CPACK_PACKAGE_NAME "onnx-trt-dev")
SET(CPACK_PACKAGE_VERSION "0.5.9")
SET(CPACK_PACKAGE_VERSION_MAJOR "0")
SET(CPACK_PACKAGE_VERSION_MINOR "5")
SET(CPACK_PACKAGE_VERSION_PATCH "9")

INCLUDE(CPack)
