# Usage:
#    $ mkdir build       # create directory for out-of-source build
#    $ cd build          # enter the build directory
#    $ cmake ..          # configure build system
#    $ make              # make libgdf
#    $ make pytest       # trigger test

PROJECT(libgdf)
cmake_minimum_required(VERSION 2.8)  # not sure about version required

find_package(CUDA)
set (CMAKE_CXX_STANDARD 11)

include_directories(include
                    $ENV{CONDA_PREFIX}/include
                    ${CMAKE_CURRENT_SOURCE_DIR}/thirdparty/cub
                    ${CMAKE_CURRENT_SOURCE_DIR}/thirdparty/moderngpu/src)

link_directories($ENV{CONDA_PREFIX}/lib)

set(CUDA_NVCC_FLAGS  ${CUDA_NVCC_FLAGS};-std=c++11;--expt-extended-lambda)

IF(CUDA_VERSION_MAJOR GREATER 7)
    set(CUDA_NVCC_FLAGS ${CUDA_NVCC_FLAGS};-gencode;arch=compute_60,code=sm_60)
ENDIF(CUDA_VERSION_MAJOR GREATER 7)

IF(CUDA_VERSION_MAJOR GREATER 8)
    set(CUDA_NVCC_FLAGS ${CUDA_NVCC_FLAGS};-gencode;arch=compute_70,code=sm_70)
ENDIF(CUDA_VERSION_MAJOR GREATER 8)

IF(CMAKE_BUILD_TYPE MATCHES DEBUG)
    set(CUDA_NVCC_FLAGS ${CUDA_NVCC_FLAGS};-g;-G)
ENDIF(CMAKE_BUILD_TYPE MATCHES DEBUG)

# switch between hash-join and sort-join
option(HASH_JOIN "hash-based join" OFF)
if(HASH_JOIN)
  message("-- Using the hash-based join implementation")
  set(CUDA_NVCC_FLAGS  ${CUDA_NVCC_FLAGS};-DHASH_JOIN)
endif()

# switch between cudaMalloc and cudaMallocManaged for hash tables
option(HT_LEGACY_ALLOCATOR "legacy allocator for hash tables" ON)
if(HT_LEGACY_ALLOCATOR)
  message("-- Using the legacy allocator for hash tables")
  set(CUDA_NVCC_FLAGS  ${CUDA_NVCC_FLAGS};-DHT_LEGACY_ALLOCATOR)
endif()

cuda_add_library(gdf SHARED
    src/binaryops.cu
    src/column.cpp
    src/errorhandling.cpp
    src/unaryops.cu
    src/ipc.cu
    src/reductions.cu
    src/sorting.cu
    src/joining.cu
    src/scan.cu
    src/segmented_sorting.cu
)

target_link_libraries(gdf arrow)

# Command to symlink files into the build directory

add_custom_command(  # link the include directory
    OUTPUT include
    COMMAND ${CMAKE_COMMAND} -E create_symlink ${CMAKE_CURRENT_SOURCE_DIR}/include include

    DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/include)

add_custom_command(  # link the python directory
    OUTPUT libgdf_cffi setup.py tests
    COMMAND ${CMAKE_COMMAND} -E create_symlink ${CMAKE_CURRENT_SOURCE_DIR}/python/libgdf_cffi libgdf_cffi
    COMMAND ${CMAKE_COMMAND} -E create_symlink ${CMAKE_CURRENT_SOURCE_DIR}/python/tests tests
    COMMAND ${CMAKE_COMMAND} -E create_symlink ${CMAKE_CURRENT_SOURCE_DIR}/python/setup.py setup.py)

add_custom_command(  # trigger cffi to build the wrapper
    OUTPUT libgdf_cffi/libgdf_cffi.py
    COMMAND python setup.py build_ext --inplace
    DEPENDS setup.py libgdf_cffi include)

add_custom_target(   # target to link the python files and trigger cffi
    copy_python
    DEPENDS libgdf_cffi/libgdf_cffi.py)

# The test target

add_custom_target(pytest DEPENDS copy_python)

add_custom_command(
    TARGET pytest POST_BUILD
    COMMAND py.test -v)

# The install target

install(TARGETS gdf
        LIBRARY DESTINATION lib)

install(DIRECTORY include/gdf DESTINATION include)
