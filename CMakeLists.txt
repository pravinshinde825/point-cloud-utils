cmake_minimum_required(VERSION 3.10)
project(point-cloud-utils)
set(CMAKE_CXX_STANDARD 14)
set(CMAKE_BUILD_TYPE "Release")

option(GEOGRAM_WITH_HLBFGS "Non-linear solver (Yang Liu's HLBFGS)"             ON)
#option(NPE_WITH_EIGEN      "Whether to use the bundled version of Eigen"       ON)
option(EIGEN_WITH_MKL      "Whether to build Eigen with intel MKL or not"      OFF)

list(APPEND CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cmake)

# Download NumpyEigen if we haven't already
set(EXTERNAL_DEP_DIR ${CMAKE_CURRENT_SOURCE_DIR}/external)
include(DownloadExternalDeps)
download_dep(numpyeigen
  GIT_REPOSITORY https://github.com/fwilliams/numpyeigen.git
  GIT_TAG c937319f70fc50cc51757556be5e6587f218b05d
)
list(APPEND CMAKE_MODULE_PATH ${EXTERNAL_DEP_DIR}/numpyeigen/cmake)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)
include(numpyeigen) # This will define Eigen3::Eigen if we enabled NPE_WITH_EIGEN

# FIXME: MKL Support is totally broken
# Optionally build Eigen with the MKL backend
#if (${EIGEN_WITH_MKL})
#  find_package(MKL REQUIRED)
#  target_link_libraries(Eigen3::Eigen INTERFACE ${MKL_LIBRARIES})
#  target_include_directories(Eigen3::Eigen INTERFACE ${MKL_INCLUDE_DIRS})
#  target_compile_definitions(Eigen3::Eigen INTERFACE EIGEN_USE_MKL_ALL)
#endif(${EIGEN_WITH_MKL})

# Build geogram
set(THIRD_PARTY_DIR ${EXTERNAL_DEP_DIR})
include(geogram)
if(${CMAKE_SYSTEM_NAME} MATCHES "Linux")
  SET_TARGET_PROPERTIES(geogram PROPERTIES COMPILE_FLAGS -fopenmp LINK_FLAGS -fopenmp)
  target_compile_options(geogram PUBLIC    -fopenmp)
  SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fopenmp")

  find_package(OpenMP REQUIRED)
  if(NOT TARGET OpenMP::OpenMP_CXX)
    add_library(OpenMP_TARGET INTERFACE)
    add_library(OpenMP::OpenMP_CXX ALIAS OpenMP_TARGET)
    target_compile_options(OpenMP_TARGET INTERFACE ${OpenMP_CXX_FLAGS})
    find_package(Threads REQUIRED)
    target_link_libraries(OpenMP_TARGET INTERFACE Threads::Threads)
    target_link_libraries(OpenMP_TARGET INTERFACE ${OpenMP_CXX_FLAGS})
  endif()
endif()

# Create the python module
npe_add_module(point_cloud_utils
  BINDING_SOURCES
  ${CMAKE_CURRENT_SOURCE_DIR}/src/sample.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/src/point_cloud_distance.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/src/lloyd.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/src/meshio.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/src/mesh_utils.cpp)
target_include_directories(point_cloud_utils PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/external/vcglib)
target_include_directories(point_cloud_utils PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/src)
target_include_directories(point_cloud_utils PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/external/nanoflann)
target_include_directories(point_cloud_utils PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/external)
target_link_libraries(point_cloud_utils PRIVATE geogram)
set_target_properties(point_cloud_utils PROPERTIES COMPILE_FLAGS "-fvisibility=hidden -msse3")

if (${CMAKE_SYSTEM_NAME} MATCHES "Linux")
  target_link_libraries(point_cloud_utils PUBLIC OpenMP::OpenMP_CXX)
endif()

