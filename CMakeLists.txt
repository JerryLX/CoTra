cmake_minimum_required(VERSION 3.0)
project(rdma-anns)

list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake/Modules")

set(CMAKE_BUILD_TYPE "Release")

message(STATUS "Build CoroMem library")

include(GNUInstallDirs)
include(CheckCXXCompilerFlag)

set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

add_compile_options(-Ofast -fpic -w -ftree-vectorize -fcoroutines)
add_compile_definitions(HAVE_CXX0X)

set(CMAKE_THREAD_LIBS_INIT "-lpthread")

find_package(Threads REQUIRED)
find_package(OpenMP REQUIRED)
find_package(BLAS REQUIRED)
find_package(LAPACK REQUIRED)
find_path(LAPACKE_INCLUDE_DIR lapacke.h REQUIRED)
find_library(LAPACKE_LIBRARY NAMES lapacke REQUIRED)
find_package(Boost 1.58.0 REQUIRED COMPONENTS serialization iostreams)
find_library(MEMCACHED_LIB memcached REQUIRED)
find_library(IBVERBS_LIB ibverbs REQUIRED)
find_library(RDMACM_LIB rdmacm REQUIRED)

include_directories(./third_party)
set(THIRD_PARTY_DIR ${CMAKE_SOURCE_DIR}/third_party)
add_subdirectory(${THIRD_PARTY_DIR})

add_custom_target(lib)
add_subdirectory(coromem)

message(STATUS "Build RDMA library")

# for rdma deps
link_libraries(Threads::Threads OpenMP::OpenMP_CXX ${IBVERBS_LIB} ${RDMACM_LIB} ${MEMCACHED_LIB})
include_directories("${PROJECT_SOURCE_DIR}")

add_library(rdma_anns)
file(GLOB sources 
  ${CMAKE_SOURCE_DIR}/src/*.cc 
  ${CMAKE_SOURCE_DIR}/src/rdma/*.cc 
  ${CMAKE_SOURCE_DIR}/src/anns/*.cc 
)

target_sources(rdma_anns PRIVATE ${sources})

target_include_directories(rdma_anns PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
  $<INSTALL_INTERFACE:include>
)


find_package(Boost COMPONENTS program_options system filesystem)

include_directories(${LAPACKE_INCLUDE_DIR})
link_libraries(${BLAS_LIBRARIES} ${LAPACK_LIBRARIES} ${LAPACKE_LIBRARY} m dl ${Boost_LIBRARIES})

add_library(rdma_index)
file(GLOB index_sources 
  ${CMAKE_SOURCE_DIR}/src/index/*.cc
)
target_sources(rdma_index PRIVATE ${index_sources})
target_include_directories(rdma_index PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
  $<INSTALL_INTERFACE:include>
)

add_subdirectory(tests)

