# SPDX-FileCopyrightText: Copyright (c) 2023 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: MIT
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
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL
# THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING
# FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER
# DEALINGS IN THE SOFTWARE.

cmake_minimum_required(VERSION 3.10)
project(centerpoint)
set(arch ${CMAKE_HOST_SYSTEM_PROCESSOR})

# JetPack 6 ships TensorRT 10 in the standard system locations.
if(${arch}     STREQUAL  "aarch64")
  set(TENSORRT_INCLUDE /usr/include/aarch64-linux-gnu)
  set(TENSORRT_LIB     /usr/lib/aarch64-linux-gnu)
elseif(${arch} STREQUAL "x86_64")
  set(TENSORRT_INCLUDE /usr/include/x86_64-linux-gnu)
  set(TENSORRT_LIB     /usr/lib/x86_64-linux-gnu)
endif()

# Local protobuf 3.21.12 prefix (matches the committed onnx *.pb.* generated sources).
set(PROTOBUF_ROOT /home/ros/pref_ws/src/protobuf-3.21)

set(CMAKE_BUILD_TYPE "Release")
set(CMAKE_CXX_FLAGS_RELEASE "-std=c++17 -Wextra -Wall -Wno-deprecated-declarations -O3")
set(CUDA_NVCC_FLAGS ${CUDA_NVCC_FLAGS} -std=c++17 -gencode arch=compute_87,code=sm_87)

# Make the bundled libspconv.so discoverable at runtime without LD_LIBRARY_PATH.
set(CMAKE_BUILD_RPATH "${CMAKE_SOURCE_DIR}/3rdparty/libspconv/lib/${arch}")

find_package(CUDA REQUIRED)
if(${CMAKE_BUILD_TYPE} STREQUAL "Debug")
  message("Using Debug Mode")
  set(CUDA_NVCC_FLAGS ${CUDA_NVCC_FLAGS} -g -G --ptxas-options=-v)
endif()

include_directories(
  ${CUDA_INCLUDE_DIRS}
  ${TENSORRT_INCLUDE}
  ${PROTOBUF_ROOT}/include
  3rdparty/libspconv/include
  3rdparty/libspconv/parser
  3rdparty/libspconv
  include
)

link_directories(
  ${TENSORRT_LIB}
  3rdparty/libspconv/lib/${arch}
)

file(GLOB_RECURSE SOURCE_FILES 
  src/*.cu 
  src/*.cpp
)

# Open-source ONNX parser + generated protobuf sources required by libspconv 1.3.x.
set(SPCONV_PARSER_SOURCES
  3rdparty/libspconv/parser/onnx-parser.cpp
  3rdparty/libspconv/onnx/onnx-ml.pb.cpp
  3rdparty/libspconv/onnx/onnx-operators-ml.pb.cpp
)

cuda_add_executable(${PROJECT_NAME} main.cpp ${SOURCE_FILES} ${SPCONV_PARSER_SOURCES})

target_link_libraries(${PROJECT_NAME}
    libnvinfer.so
    libspconv.so
    ${PROTOBUF_ROOT}/lib/libprotobuf.a
    z
    pthread
)