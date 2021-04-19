cmake_minimum_required(VERSION 3.4.1)
set(TARGET wenet)
project(${TARGET} CXX)
set(CMAKE_CXX_STANDARD 14)
include(ExternalProject)

include_directories(${CMAKE_SOURCE_DIR})
set(CMAKE_VERBOSE_MAKEFILE on)

# Pytorch
set(build_DIR ${CMAKE_SOURCE_DIR}/../../../build)
file(GLOB PYTORCH_INCLUDE_DIRS "${build_DIR}/pytorch_android*.aar/headers")
file(GLOB PYTORCH_LINK_DIRS "${build_DIR}/pytorch_android*.aar/jni/${ANDROID_ABI}")
find_library(PYTORCH_LIBRARY pytorch_jni
  PATHS ${PYTORCH_LINK_DIRS}
  NO_CMAKE_FIND_ROOT_PATH
)
find_library(FBJNI_LIBRARY fbjni
  PATHS ${PYTORCH_LINK_DIRS}
  NO_CMAKE_FIND_ROOT_PATH
)
include_directories(
  ${PYTORCH_INCLUDE_DIRS}
  ${PYTORCH_INCLUDE_DIRS}/torch/csrc/api/include
)

set(ANDROID_CMAKE_ARGS
  -DCMAKE_TOOLCHAIN_FILE=${CMAKE_TOOLCHAIN_FILE}
  -DANDROID_NDK=${ANDROID_NDK}
  -DANDROID_ARM_NEON=ON
  -DANDROID_ABI=${ANDROID_ABI}
  -DCMAKE_BUILD_TYPE=Release
  -DCMAKE_MAKE_PROGRAM=${CMAKE_MAKE_PROGRAM}
  -DANDROID_NATIVE_API_LEVEL=${ANDROID_NATIVE_API_LEVEL}
  -DANDROID_PLATFORM=${ANDROID_PLATFORM}
  -DCMAKE_CXX_FLAGS=${CMAKE_CXX_FLAGS}
  -DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX}
  -DBUILD_TESTING=OFF
)

# third party: gflags
set(gflags_SRC_DIR ${build_DIR}/gflags-src)
set(gflags_BUILD_DIR ${CMAKE_BINARY_DIR}/gflags-build)
set(gflags_LIB ${gflags_BUILD_DIR}/lib/libgflags_nothreads.a)
ExternalProject_Add(gflags
  URL              https://github.com/gflags/gflags/archive/v2.2.0.zip
  URL_HASH         SHA256=99f9e8b63ea53e4b23f0fd0f68a6f1e397d5512be36716c17cc75966a90f0d57
  SOURCE_DIR       ${gflags_SRC_DIR}
  BINARY_DIR       ${gflags_BUILD_DIR}
  BUILD_BYPRODUCTS ${gflags_LIB}
  CMAKE_ARGS       ${ANDROID_CMAKE_ARGS}
  INSTALL_COMMAND  ""
)
include_directories(${gflags_BUILD_DIR}/include)
link_libraries(${gflags_LIB})

# third party: glog
set(glog_SRC_DIR ${build_DIR}/glog-src)
set(glog_BUILD_DIR ${CMAKE_BINARY_DIR}/glog-build)
set(glog_LIB ${glog_BUILD_DIR}/libglog.a)
ExternalProject_Add(glog
  URL              https://github.com/google/glog/archive/v0.4.0.zip
  URL_HASH         SHA256=9e1b54eb2782f53cd8af107ecf08d2ab64b8d0dc2b7f5594472f3bd63ca85cdc
  SOURCE_DIR       ${glog_SRC_DIR}
  BINARY_DIR       ${glog_BUILD_DIR}
  BUILD_BYPRODUCTS ${glog_LIB}
  CMAKE_ARGS       ${ANDROID_CMAKE_ARGS}
  INSTALL_COMMAND  ""
)
include_directories(${glog_SRC_DIR}/src)
include_directories(${glog_BUILD_DIR})
link_libraries(${glog_LIB})

# third_party: openfst
# openfst is compiled with glog/gflags to avoid log and flag conflicts with log and flags in wenet/libtorch
set(openfst_SRC_DIR ${build_DIR}/openfst-src)
set(openfst_BUILD_DIR ${CMAKE_BINARY_DIR}/openfst-build)
ExternalProject_Add(openfst
  URL               https://github.com/mjansche/openfst/archive/1.6.5.zip
  URL_HASH          SHA256=b720357a464f42e181d7e33f60867b54044007f50baedc8f4458a3926f4a5a78
  SOURCE_DIR        ${openfst_SRC_DIR}
  BINARY_DIR        ${openfst_BUILD_DIR}
  CONFIGURE_COMMAND ${openfst_SRC_DIR}/configure --host=${CMAKE_LIBRARY_ARCHITECTURE} --disable-bin
                        "LIBS=${gflags_LIB} ${glog_LIB}"
  COMMAND           cp ${CMAKE_SOURCE_DIR}/patch/openfst/log.h ${openfst_SRC_DIR}/src/include/fst
  COMMAND           cp ${CMAKE_SOURCE_DIR}/patch/openfst/flags.h ${openfst_SRC_DIR}/src/include/fst
  COMMAND           cp ${CMAKE_SOURCE_DIR}/patch/openfst/flags.cc ${openfst_SRC_DIR}/src/lib
  BUILD_COMMAND     make -j 4
  INSTALL_COMMAND   ""
)
include_directories(${openfst_SRC_DIR}/src/include)
link_directories(${openfst_BUILD_DIR}/src/lib/.libs)
add_dependencies(openfst gflags glog)

# utils
add_library(utils STATIC utils/utils.cc)
add_dependencies(utils openfst)
target_link_libraries(utils PUBLIC fst log)

# frontend
add_library(frontend STATIC
  frontend/feature_pipeline.cc
  frontend/fft.cc
)
target_link_libraries(frontend PUBLIC utils)

# decoder
add_library(decoder STATIC
  decoder/ctc_endpoint.cc
  decoder/ctc_prefix_beam_search.cc
  decoder/torch_asr_decoder.cc
  decoder/torch_asr_model.cc
)
target_link_libraries(decoder PUBLIC ${PYTORCH_LIBRARY} ${FBJNI_LIBRARY} utils)

link_libraries(utils frontend decoder android)
add_library(${TARGET} SHARED wenet.cc)

add_executable(decoder_main bin/decoder_main.cc)
target_link_libraries(decoder_main PUBLIC libc++_shared.so)
