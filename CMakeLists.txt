cmake_minimum_required(VERSION 3.0.2)

project(mxnet C CXX)

if(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/build/private/local_config.cmake)
  include(${CMAKE_CURRENT_SOURCE_DIR}/build/private/local_config.cmake)
endif()

set(CMAKE_MODULE_PATH "${PROJECT_SOURCE_DIR}/cmake/Modules;${CMAKE_MODULE_PATH}")

include(cmake/Utils.cmake)
mxnet_option(USE_OPENCV           "Build with OpenCV support" ON)
mxnet_option(USE_OPENMP           "Build with Openmp support" ON)
mxnet_option(USE_CUDA             "Build with CUDA support"   ON)
mxnet_option(USE_CUDNN            "Build with cudnn support"  ON) # one could set CUDNN_ROOT for search path
mxnet_option(USE_MKL_IF_AVAILABLE "Use MKL if found" ON)
mxnet_option(USE_MKLML_MKL        "Use MKLML variant of MKL (if MKL found)" ON IF USE_MKL_IF_AVAILABLE AND UNIX AND (NOT APPLE))
mxnet_option(USE_MKL_EXPERIMENTAL "Use experimental MKL (if MKL enabled and found)" OFF)
mxnet_option(USE_JEMALLOC         "Build with Jemalloc support"   OFF)
mxnet_option(USE_PROFILER         "Build with Profiler support"   OFF)
mxnet_option(USE_DIST_KVSTORE     "Build with DIST_KVSTORE support" OFF)
mxnet_option(USE_PLUGINS_WARPCTC	"Use WARPCTC Plugins" OFF)
mxnet_option(USE_PLUGIN_CAFFE     "Use Caffe Plugin" OFF)
mxnet_option(USE_CPP_PACKAGE      "Build C++ Package" OFF)
mxnet_option(USE_MXNET_LIB_NAMING "Use MXNet library naming conventions." ON)

SET(EXTRA_OPERATORS "" CACHE PATH "EXTRA OPERATORS PATH")

if("$ENV{VERBOSE}" STREQUAL "1")
  message(STATUS " Verbose Makefile ACTIVATED")
  set(CMAKE_VERBOISE_MAKEFILE ON)
endif()


if(MSVC)
  add_definitions(-DWIN32_LEAN_AND_MEAN)
  add_definitions(-DDMLC_USE_CXX11)
  add_definitions(-DMSHADOW_IN_CXX11)
  add_definitions(-D_SCL_SECURE_NO_WARNINGS)
  add_definitions(-D_CRT_SECURE_NO_WARNINGS)
  add_definitions(-DMXNET_EXPORTS)
  add_definitions(-DNNVM_EXPORTS)
  add_definitions(-DDMLC_STRICT_CXX11)
  set(CMAKE_C_FLAGS "/MP")
  set(CMAKE_CXX_FLAGS "${CMAKE_C_FLAGS} /bigobj")
else(MSVC)
  include(CheckCXXCompilerFlag)
  check_cxx_compiler_flag("-std=c++11"   SUPPORT_CXX11)
  check_cxx_compiler_flag("-msse2"       SUPPORT_MSSE2)
  set(CMAKE_C_FLAGS "-Wall -Wno-unknown-pragmas -fPIC")
  if(NDEBUG)
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -O3")
  else(NDEBUG)
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -O0 -ggdb3")
  endif(NDEBUG)
  set(CMAKE_CXX_FLAGS ${CMAKE_C_FLAGS})
  if(SUPPORT_CXX11)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
  endif()
  if(SUPPORT_MSSE2)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -msse2")
  endif()
endif(MSVC)

set(mxnet_LINKER_LIBS "")

if(USE_MKL_IF_AVAILABLE)
  if(USE_MKL_EXPERIMENTAL AND NOT USE_MKLML_MKL)
    message(ERROR " USE_MKL_EXPERIMENTAL can only be used when USE_MKL_EXPERIMENTAL is enabled")
  endif()
  find_package(MKL)
  if(MKL_FOUND)
    include_directories(${MKL_INCLUDE_DIR})
    include_directories(${CMAKE_CURRENT_SOURCE_DIR}/src/operator/mkl)
    add_definitions(-DMXNET_USE_MKL2017=1)
    add_definitions(-DUSE_MKL=1)
    add_definitions(-DCUB_MKL=1)
    list(APPEND mxnet_LINKER_LIBS ${MKL_LIBRARIES})
    if(NOT MSVC)
      list(APPEND mxnet_LINKER_LIBS dl)
    endif()
    if(USE_MKL_EXPERIMENTAL)
      add_definitions(-DMKL_EXPERIMENTAL=1)
    else()
      add_definitions(-DMKL_EXPERIMENTAL=0)
    endif()
  else()
    message(STATUS " MKL not found")
  endif()
endif()

# Allow Cuda compiles outside of src tree to find things in 'src' and 'include'
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/include)
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/src)

if(EXISTS ${PROJECT_SOURCE_DIR}/mshadow/cmake)
  include(mshadow/cmake/mshadow.cmake)
  include(mshadow/cmake/Utils.cmake)
  include(mshadow/cmake/Cuda.cmake)
else()
  include(mshadowUtils)
  include(Cuda)
  include(mshadow)
endif()

list(APPEND mxnet_LINKER_LIBS ${mshadow_LINKER_LIBS})

foreach(var ${C_CXX_INCLUDE_DIRECTORIES})
 	include_directories(${var})
endforeach()

include_directories("include")
include_directories("mshadow")
include_directories("cub")
include_directories("nnvm/include")
include_directories("dmlc-core/include")

if(NOT MSVC)
  set(BEGIN_WHOLE_ARCHIVE -Wl,--whole-archive)
  set(END_WHOLE_ARCHIVE -Wl,--no-whole-archive)
endif()

if(UNIX)
  find_library(RTLIB rt)
  if(RTLIB)
    list(APPEND mxnet_LINKER_LIBS ${RTLIB})
  endif()
endif()

# ---[ jemalloc
if(USE_JEMALLOC)
  find_package(JeMalloc)
  if(JEMALLOC_FOUND)
    add_definitions(-DUSE_JEMALLOC)
    include_directories(${JEMALLOC_INCLUDE_DIRS})
    set(mxnet_LINKER_LIBS ${mxnet_LINKER_LIBS} ${JEMALLOC_LIBRARIES})
  endif()
endif()

if(USE_OPENCV)
  find_package(OpenCV QUIET COMPONENTS core highgui imgproc imgcodecs)
  if(NOT OpenCV_FOUND) # if not OpenCV 3.x, then imgcodecs are not found
    find_package(OpenCV REQUIRED COMPONENTS core highgui imgproc)
  endif()
  include_directories(SYSTEM ${OpenCV_INCLUDE_DIRS})
  list(APPEND mxnet_LINKER_LIBS ${OpenCV_LIBS})
  message(STATUS " OpenCV_LIBS=${OpenCV_LIBS}")
  message(STATUS "OpenCV found (${OpenCV_CONFIG_PATH})")
  add_definitions(-DMXNET_USE_OPENCV=1)
  if(NOT MSVC)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wl,--no-undefined")
  endif()
else(USE_OPENCV)
  message(STATUS "OpenCV Disabled")
  add_definitions(-DMXNET_USE_OPENCV=0)
endif()

if(USE_OPENMP)
  find_package(OpenMP REQUIRED)
  if(OPENMP_FOUND)
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${OpenMP_C_FLAGS}")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
    set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} ${OpenMP_EXE_LINKER_FLAGS}")
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} ${OpenMP_EXE_LINKER_FLAGS}")
  endif()
endif()

if(UNIX)
  find_library(RTLIB rt)
  if(RTLIB)
    list(APPEND mxnet_LINKER_LIBS ${RTLIB})
  endif()
endif()

# ---[ jemalloc
if(USE_JEMALLOC)
  find_package(JeMalloc)
  if(JEMALLOC_FOUND)
    add_definitions(-DUSE_JEMALLOC)
    include_directories(${JEMALLOC_INCLUDE_DIRS})
    set(mxnet_LINKER_LIBS ${mxnet_LINKER_LIBS} ${JEMALLOC_LIBRARIES})
  endif()
endif()

include(CTest)

# cudnn detection
if(USE_CUDNN AND USE_CUDA)
  detect_cuDNN()
  if(HAVE_CUDNN)
    add_definitions(-DUSE_CUDNN)
    include_directories(SYSTEM ${CUDNN_INCLUDE})
    list(APPEND mxnet_LINKER_LIBS ${CUDNN_LIBRARY})
	  add_definitions(-DMSHADOW_USE_CUDNN=1)
  endif()
endif()

if(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/dmlc-core/cmake)
  add_subdirectory("dmlc-core")
endif()


mxnet_source_group("Include\\common"   GLOB "src/common/*.h")
mxnet_source_group("Include\\c_api"   GLOB "src/c_api/*.h")
mxnet_source_group("Include\\engine"   GLOB "src/engine/*.h")
mxnet_source_group("Include\\io"   GLOB "src/io/*.h")
mxnet_source_group("Include\\kvstore"   GLOB "src/kvstore/*.h")
mxnet_source_group("Include\\ndarray"   GLOB "src/ndarray/*.h")
mxnet_source_group("Include\\operator"   GLOB "src/operator/*.h")
mxnet_source_group("Include\\operator\\tensor"   GLOB "src/operator/tensor/*.h")
mxnet_source_group("Include\\optimizer"   GLOB "src/optimizer/*.h")
mxnet_source_group("Include\\resource.cc"   GLOB "src/resource.cc/*.h")
mxnet_source_group("Include\\storage"   GLOB "src/storage/*.h")
mxnet_source_group("Include\\symbol"   GLOB "src/symbol/*.h")
mxnet_source_group("Include\\executor"   GLOB "src/executor/*.h")

mxnet_source_group("Cuda\\common"   GLOB "src/common/*.cu")
mxnet_source_group("Cuda\\c_api"   GLOB "src/c_api/*.cu")
mxnet_source_group("Cuda\\engine"   GLOB "src/engine/*.cu")
mxnet_source_group("Cuda\\io"   GLOB "src/io/*.cu")
mxnet_source_group("Cuda\\kvstore"   GLOB "src/kvstore/*.cu")
mxnet_source_group("Cuda\\ndarray"   GLOB "src/ndarray/*.cu")
mxnet_source_group("Cuda\\operator"   GLOB "src/operator/*.cu")
mxnet_source_group("Cuda\\operator\\tensor"   GLOB "src/operator/tensor/*.cu"  "src/operator/tensor/*.cuh")
mxnet_source_group("Cuda\\optimizer"   GLOB "src/optimizer/*.cu")
mxnet_source_group("Cuda\\resource.cc"   GLOB "src/resource.cc/*.cu")
mxnet_source_group("Cuda\\storage"   GLOB "src/storage/*.cu")
mxnet_source_group("Cuda\\symbol"   GLOB "src/symbol/*.cu")

mxnet_source_group("Source"   GLOB "src/*.cc")
mxnet_source_group("Source\\common"   GLOB "src/common/*.cc")
mxnet_source_group("Source\\c_api"   GLOB "src/c_api/*.cc")
mxnet_source_group("Source\\engine"   GLOB "src/engine/*.cc")
mxnet_source_group("Source\\io"   GLOB "src/io/*.cc")
mxnet_source_group("Source\\kvstore"   GLOB "src/kvstore/*.cc")
mxnet_source_group("Source\\ndarray"   GLOB "src/ndarray/*.cc")
mxnet_source_group("Source\\operator"   GLOB "src/operator/*.cc")
mxnet_source_group("Source\\operator\\tensor"   GLOB "src/operator/tensor/*.cc")
mxnet_source_group("Source\\optimizer"   GLOB "src/optimizer/*.cc")
mxnet_source_group("Source\\resource.cc"   GLOB "src/resource.cc/*.cc")
mxnet_source_group("Source\\storage"   GLOB "src/storage/*.cc")
mxnet_source_group("Source\\symbol"   GLOB "src/symbol/*.cc")
mxnet_source_group("Source\\executor"   GLOB "src/executor/*.cc")
mxnet_source_group("Source\\nnvm"   GLOB "src/nnvm/*.cc")


FILE(GLOB_RECURSE SOURCE "src/*.cc" "src/*.h")
FILE(GLOB_RECURSE CUDA "src/*.cu" "src/*.cuh")

# add nnvm to source

mxnet_source_group("NNVM\\Include\\c_api"   GLOB "nnvm/src/c_api/*.h")
mxnet_source_group("NNVM\\Include\\core"   GLOB "nnvm/src/core/*.h")
mxnet_source_group("NNVM\\Include\\pass"   GLOB "nnvm/src/pass/*.h")
mxnet_source_group("NNVM\\Include\\nnvm"   GLOB "nnvm/include/nnvm/*.h")
mxnet_source_group("NNVM\\Include\\dmlc"   GLOB "nnvm/include/dmlc/*.h")

mxnet_source_group("NNVM\\Source"   GLOB "nnvm/src/*.cc")
mxnet_source_group("NNVM\\Source\\c_api"   GLOB "nnvm/src/c_api/*.cc")
mxnet_source_group("NNVM\\Source\\core"   GLOB "nnvm/src/core/*.cc")
mxnet_source_group("NNVM\\Source\\pass"   GLOB "nnvm/src/pass/*.cc")

FILE(GLOB_RECURSE NNVMSOURCE "nnvm/src/*.cc" "nnvm/src/*.h" "nnvm/include/*.h")
list(APPEND SOURCE ${NNVMSOURCE})

# add mshadow file
mxnet_source_group("mshadow\\Include"   GLOB "mshadow/mshadow/*.h")
mxnet_source_group("mshadow\\Include\\cuda"   GLOB "mshadow/mshadow/cuda/*.h" "mshadow/mshadow/cuda/*.cuh")
mxnet_source_group("mshadow\\Include\\extension"   GLOB "mshadow/mshadow/extension/*.h")
mxnet_source_group("mshadow\\Include\\packet"   GLOB "mshadow/mshadow/packet/*.h")

FILE(GLOB_RECURSE MSHADOWSOURCE "mshadow/mshadow/*.h")
FILE(GLOB_RECURSE MSHADOW_CUDASOURCE "mshadow/mshadow/*.cuh")
list(APPEND SOURCE ${MSHADOWSOURCE})
list(APPEND CUDA ${MSHADOW_CUDASOURCE})

if(USE_PLUGINS_WARPCTC)
	set(WARPCTC_INCLUDE  "" CACHE PATH "WARPCTC include")
    set(WARPCTC_LIB_DEBUG  "" CACHE FILEPATH "WARPCTC lib")
    set(WARPCTC_LIB_RELEASE  "" CACHE FILEPATH "WARPCTC lib")


	include_directories(SYSTEM ${WARPCTC_INCLUDE})
	list(APPEND mxnet_LINKER_LIBS ${WARPCTC_LIB})
	mxnet_source_group("Include\\plugin\\warpctc"   GLOB "plugin/warpctc/*.h")
	mxnet_source_group("Source\\plugin\\warpctc"   GLOB "plugin/warpctc/*.cc")
	mxnet_source_group("Cuda\\plugin\\warpctc"   GLOB "plugin/warpctc/*.cu")
	FILE(GLOB_RECURSE PLUGINS_SOURCE "plugin/warpctc/*.cc" "plugin/warpctc/*.h")
	FILE(GLOB_RECURSE PLUGINS_CUSRC "plugin/warpctc/*.cu")
	list(APPEND SOURCE ${PLUGINS_SOURCE})
	list(APPEND CUDA ${PLUGINS_CUSRC})
endif()

if(USE_PLUGIN_CAFFE)
  if(NOT USE_CUDA)
    set(CPU_ONLY ON)
    add_definitions(-DCPU_ONLY=1)
  endif()
  if(NOT DEFINED CAFFE_PATH)
    if(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/caffe)
      # Need newer FindCUDA.cmake that correctly handles -std=c++11
      cmake_minimum_required(VERSION 3.3)
      set(CAFFE_PATH ${PROJECT_SOURCE_DIR}/caffe)
    endif()
  endif()
  list(APPEND CMAKE_MODULE_PATH ${CAFFE_PATH}/cmake)
  include_directories(${CAFFE_PATH}/include)
  include_directories(${CAFFE_PATH}/build/src)
  include_directories(${CMAKE_BINARY_DIR}/caffe/include)
  link_directories(${CAFFE_PATH}/build/lib)
  if(NOT DEFINED CAFFE_PATH)
    message(FATAL_ERROR "Please set CAFFE_PATH to point to the caffe source installation")
  endif()
  mxnet_source_group("Include\\plugin\\caffe"   GLOB "plugin/caffe/*.h")
  mxnet_source_group("Source\\plugin\\caffe"   GLOB "plugin/caffe/*.cc")
  mxnet_source_group("Cuda\\plugin\\caffe"   GLOB "plugin/caffe/*.cu")
  FILE(GLOB_RECURSE PLUGINS_SOURCE "plugin/caffe/*.cc" "plugin/caffe/*.h")
  FILE(GLOB_RECURSE PLUGINS_CUSRC "plugin/caffe/*.cu")
  list(APPEND SOURCE ${PLUGINS_SOURCE})
  list(APPEND CUDA ${PLUGINS_CUSRC})
  include_directories(${CMAKE_BINARY_DIR}/include)
  add_definitions(-DMXNET_USE_CAFFE=1)
  list(APPEND mxnet_LINKER_LIBS
    protobuf boost_system boost_thread boost_filesystem
    gflags glog caffe
    ${Caffe_LINKER_LIBS}
)
endif()

if (NOT (EXTRA_OPERATORS STREQUAL ""))
	mxnet_source_group("Extra"   GLOB_RECURSE "${EXTRA_OPERATORS}/*.cc")
	mxnet_source_group("Extra\\Cuda"   GLOB_RECURSE "${EXTRA_OPERATORS}/*.cu")
	FILE(GLOB_RECURSE EXTRA_SRC "${EXTRA_OPERATORS}/*.cc")
	FILE(GLOB_RECURSE EXTRA_CUSRC "${EXTRA_OPERATORS}/*.cu")
	list(APPEND SOURCE ${EXTRA_SRC} ${EXTRA_CUSRC})
endif()

if(MSVC)
  foreach(flag_var
        CMAKE_CXX_FLAGS CMAKE_CXX_FLAGS_DEBUG CMAKE_CXX_FLAGS_RELEASE
        CMAKE_CXX_FLAGS_MINSIZEREL CMAKE_CXX_FLAGS_RELWITHDEBINFO)
    if(${flag_var} MATCHES "/MD")
      string(REGEX REPLACE "/MD" "/MT" ${flag_var} "${${flag_var}}")
    endif(${flag_var} MATCHES "/MD")
  endforeach(flag_var)
endif()

if(USE_CUDA)
  list(APPEND CUDA_INCLUDE_DIRS ${INCLUDE_DIRECTORIES})
  # define preprocessor macro so that we will not include the generated forcelink header
  mshadow_cuda_compile(cuda_objs ${CUDA})
  if(MSVC)
    FIND_LIBRARY(CUDA_nvrtc_LIBRARY nvrtc "${CUDA_TOOLKIT_ROOT_DIR}/lib/x64"  "${CUDA_TOOLKIT_ROOT_DIR}/lib/win32")
    list(APPEND mxnet_LINKER_LIBS ${CUDA_nvrtc_LIBRARY})
    set(CUDA_cuda_LIBRARY "${CUDA_nvrtc_LIBRARY}/../cuda.lib")
    list(APPEND mxnet_LINKER_LIBS ${CUDA_cuda_LIBRARY})
    FIND_LIBRARY(CUDA_cufft_LIBRARY nvrtc "${CUDA_TOOLKIT_ROOT_DIR}/lib/x64"  "${CUDA_TOOLKIT_ROOT_DIR}/lib/win32")
    list(APPEND mxnet_LINKER_LIBS "${CUDA_cufft_LIBRARY}/../cufft.lib") # For fft operator
  else(MSVC)
    list(APPEND mxnet_LINKER_LIBS nvrtc cuda cufft)
    link_directories("${CUDA_TOOLKIT_ROOT_DIR}/lib64")
  endif()
  list(APPEND SOURCE ${cuda_objs} ${CUDA})
  add_definitions(-DMXNET_USE_CUDA=1)
  add_definitions(-DMXNET_USE_NVRTC=1)
  if(CUDA_LIBRARY_PATH)
    if(IS_CONTAINER_BUILD)
      # In case of building on a production-like build container which may not have Cuda installed
      if(NOT CMAKE_SYSTEM_HAS_CUDA)
        # Assuming building in a container that doesn't have CUDA installed (ie CPU-only build machine)
        # so use the stub cuda driver shared library
        if(EXISTS ${CUDA_LIBRARY_PATH}/stubs/libcuda.so)
          link_directories(${CUDA_LIBRARY_PATH}/stubs)
        endif()
      endif()
    endif()
  endif()
endif()

# unsupported: if caffe is a subdirectory of mxnet, load its CMakeLists.txt as well
if(USE_PLUGIN_CAFFE)
  if(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/caffe)
    add_subdirectory(caffe)
  endif()
endif()

if(NOT EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/nnvm/CMakeLists.txt")
  set(nnvm_LINKER_LIBS nnvm)
  list(APPEND mxnet_LINKER_LIBS ${nnvm_LINKER_LIBS})
endif()

if(NOT MSVC)
  # Only add c++11 flags and definitions after cuda compiling
  add_definitions(-DDMLC_USE_CXX11)
  add_definitions(-DMSHADOW_IN_CXX11)
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS}")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++0x")
else()
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /EHsc")
endif()

if(${CMAKE_SYSTEM_NAME} STREQUAL "Darwin" AND USE_MXNET_LIB_NAMING)
  add_library(mxnet MODULE ${SOURCE})
else()
  if(UNIX)
    set(MXNET_DYNAMIC_ONLY ON)
    if(MXNET_DYNAMIC_ONLY)
      add_library(mxnet SHARED ${SOURCE})
    else()
      set(INITIALIZE_SOURCE_FILE ${CMAKE_CURRENT_SOURCE_DIR}/src/initialize.cc)
      list(REMOVE_ITEM SOURCE ${INITIALIZE_SOURCE_FILE})
      add_library(mxnet_static STATIC ${INITIALIZE_SOURCE_FILE} ${SOURCE})
      # Need an arbitrary source file to trigger CMake to build the library
      add_library(mxnet SHARED ${INITIALIZE_SOURCE_FILE})
      # This has prolems, as it adds libmxnet_static to INTERFACE_LINK_LIBRARIES
      target_link_libraries(mxnet "-Wl,--whole-archive $<TARGET_FILE:mxnet_static> -Wl,--no-whole-archive")
      #target_link_libraries(mxnet mxnet_static)
      add_custom_target(
          StaticallyLinkStaticMXNetLibrary ALL
          BYPRODUCTS ${CMAKE_ARCHIVE_OUTPUT_DIRECTORY}/libmxnet.a
          WORKING_DIRECTORY ${CMAKE_ARCHIVE_OUTPUT_DIRECTORY}
          COMMAND ln -sf libmxnet_static.a libmxnet.a
          DEPENDS mxnet_static
      )
    endif()
  else()
    add_library(mxnet SHARED ${SOURCE})
  endif()
endif()
target_link_libraries(mxnet ${mxnet_LINKER_LIBS})

if(USE_PLUGINS_WARPCTC)
  target_link_libraries(mxnet debug ${WARPCTC_LIB_DEBUG})
  target_link_libraries(mxnet optimized ${WARPCTC_LIB_RELEASE})
endif()

target_link_libraries(mxnet dmlc)

if(MSVC AND USE_MXNET_LIB_NAMING)
  set_target_properties(mxnet PROPERTIES OUTPUT_NAME "libmxnet")
endif()


if(USE_DIST_KVSTORE)
  if(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/ps-lite/CMakeLists.txt)
    add_subdirectory("ps-lite")
    list(APPEND pslite_LINKER_LIBS pslite)
    target_link_libraries(mxnet debug ${pslite_LINKER_LIBS_DEBUG})
    target_link_libraries(mxnet optimized ${pslite_LINKER_LIBS_RELEASE})
  else()
    set(pslite_LINKER_LIBS protobuf zmq-static )
  endif()
  add_definitions(-DMXNET_USE_DIST_KVSTORE)
  target_link_libraries(mxnet ${pslite_LINKER_LIBS})
  include_directories(SYSTEM ${pslite_INCLUDE_DIR})
endif()

if(USE_PROFILER)
	add_definitions(-DMXNET_USE_PROFILER)
endif()

# Do tests after chrpath so that we use the "real" cuda driver
add_subdirectory(tests)

# AUTO_INSTALL_DIR -> Optional: specify post-build install direcory
if(AUTO_INSTALL_DIR)
  # ---[ Install Includes
  add_custom_command(TARGET mxnet POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E copy_directory
    ${PROJECT_SOURCE_DIR}/include ${AUTO_INSTALL_DIR}/include
    )

  # ---[ Install Examples
  add_custom_command(TARGET mxnet POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E copy_directory
    ${PROJECT_SOURCE_DIR}/example ${AUTO_INSTALL_DIR}/example
    )
endif()

if(INSTALL_PYTHON_VERSIONS)
  message(STATUS "Installing for python versions: ${INSTALL_PYTHON_VERSIONS}")
  foreach(version ${INSTALL_PYTHON_VERSIONS})
    set(outdir ${CMAKE_LIBRARY_OUTPUT_DIRECTORY}/python${version}/site-packages/mxnet)
    add_custom_command(TARGET mxnet POST_BUILD
      COMMAND mkdir -p ${outdir}
      COMMAND cp -ru ${CMAKE_CURRENT_SOURCE_DIR}/python/mxnet/* ${outdir}
      )
  endforeach()
endif()

if(USE_CPP_PACKAGE)
  add_subdirectory(cpp-package)
endif()

add_subdirectory(example/image-classification/predict-cpp)

# ---[ Linter target
if(MSVC)
  find_package(PythonInterp)
  set(PYTHON_EXECUTABLE ${PYTHON_EXECUTABLE} CACHE FILEPATH "Path to the python executable")
endif()
set(LINT_DIRS include src scripts python)
add_custom_target(mxnet_lint COMMAND ${CMAKE_COMMAND} -DMSVC=${MSVC} -DPYTHON_EXECUTABLE=${PYTHON_EXECUTABLE} -DLINT_DIRS=${LINT_DIRS} -DPROJECT_SOURCE_DIR=${PROJECT_SOURCE_DIR} -DPROJECT_NAME=mxnet -P ${PROJECT_SOURCE_DIR}/dmlc-core/cmake/lint.cmake)

