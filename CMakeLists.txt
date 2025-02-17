# Copyright 2021-2023 Lawrence Livermore National Security, LLC and other
# AMSLib Project Developers
#
# SPDX-License-Identifier: Apache-2.0 WITH LLVM-exception

cmake_minimum_required(VERSION 3.18)
project(AMS VERSION 0.1.0 LANGUAGES CXX C)
cmake_policy(SET CMP0074 NEW)

set(CMAKE_CXX_STANDARD 14)

# Enable this to get compile_commands.json
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

if (NOT EXISTS ${CMAKE_BINARY_DIR}/CMakeCache.txt)
  if (NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE "Release" CACHE STRING "" FORCE)
  endif()
endif()

set(CMAKE_MODULE_PATH "${PROJECT_SOURCE_DIR}/cmake;${CMAKE_MODULE_PATH};")

set(AMS_APP_LIBRARIES "")
set(AMS_APP_DEFINES "")
set(AMS_APP_INCLUDES "")

set(AMS_EXAMPLE_LIBRARIES "")
set(AMS_EXAMPLE_DEFINES "")
set(AMS_EXAMPLE_INCLUDES "")

# ------------------------------------------------------------------------------
option(WITH_CUDA           "Option to enable CUDA" OFF)
option(WITH_EXAMPLES       "Build examples" OFF)
option(WITH_MPI            "Option to enable MPI" OFF)
option(WITH_CALIPER        "Use Caliper for Profiling" OFF)
option(WITH_FAISS          "Use C/C++ FAISS interface for HD Cache" OFF)
option(WITH_TORCH          "Use C/C++ Torch interface for Surrogate Model Inference" OFF)
option(WITH_TORCH_DEBUG    "Compute RMSE of Surrogate Model and Physics Module" OFF)
option(WITH_TESTS          "Compile tests" OFF)
option(WITH_REDIS          "Use REDIS as a database back end" OFF)
option(WITH_HDF5           "Use HDF5 as a database back end" OFF)
option(WITH_RMQ            "Use RabbitMQ as a database back end (require a reachable and running RabbitMQ server service)" OFF)
option(WITH_AMS_DEBUG      "Enable verbose messages" OFF)
option(WITH_PERFFLOWASPECT "Use PerfFlowAspect for Profiling" OFF)
option(WITH_WORKFLOW       "Install python drivers used by the outer workflow" OFF)
option(BUILD_SHARED_LIBS "Build using shared libraries" ON)

if (WITH_MPI)
  #  SET(CMAKE_CXX_COMPILER "${MPI_CXX_COMPILER}" CACHE FILEPATH "CXX compiler overridden with MPI C++ wrapper")
  #SET(CMAKE_C_COMPILER "${MPI_C_COMPILER}" CACHE FILEPATH "CXX compiler overridden with MPI C++ wrapper")

  find_package(MPI REQUIRED)
  message(STATUS "MPICC:  ${MPI_C_COMPILER}")
  message(STATUS "MPICXX: ${MPI_CXX_COMPILER}")
  list(APPEND AMS_APP_LIBRARIES MPI::MPI_CXX)
  message(STATUS "MPI Library used: " MPI::MPI_CXX)
  list(APPEND AMS_APP_DEFINES "-D__ENABLE_MPI__")
endif()

# ------------------------------------------------------------------------------
if (WITH_CUDA)
  if (NOT DEFINED AMS_CUDA_ARCH)
    message(WARNING "CUDA ARCH IS NOT DEFINED, USING 70")
    set(AMS_CUDA_ARCH 70)
  endif()

  if (BUILD_SHARED_LIBS)
    set(CUDA_RUNTIME_LIBRARY "Shared")
  else()
    set(CUDA_RUNTIME_LIBRARY "Static")
  endif()

  # we need to enable nvcc
  enable_language(CUDA)
  set(CMAKE_CUDA_STANDARD 14)
  # Turn off message of mfem
  set(ENABLE_CUDA True)
  list(APPEND AMS_APP_LIBRARIES ${CUDA_LIBRARIES} cuda)
  list(APPEND AMS_APP_DEFINES "-D__ENABLE_CUDA__")
  set(THRUST_IGNORE_CUB_VERSION_CHECK True)
endif()

# ------------------------------------------------------------------------------
if (WITH_CALIPER)
  find_package(caliper REQUIRED)
  list(APPEND AMS_APP_INCLUDES ${caliper_INCLUDE_DIR})
  list(APPEND AMS_APP_LIBRARIES caliper)
  list(APPEND AMS_APP_DEFINES "-D__ENABLE_CALIPER__")
endif()

if (WITH_AMS_DEBUG)
  list(APPEND AMS_APP_DEFINES "-DLIBAMS_VERBOSE")
endif()

# ------------------------------------------------------------------------------
list(APPEND AMS_APP_DEFINES "-D__ENABLE_DB__")

if (WITH_REDIS)
  # Temprorary fix for:= the following error which happens when using Cuda 11.6 and Redis backend
  # error: #error The version of CUB in your include path is not compatible with this release of Thrust.
  if (WITH_CUDA)
    add_compile_definitions(THRUST_IGNORE_CUB_VERSION_CHECK)
  endif()
  if (HIREDIS_DIR)
    find_path(HIREDIS_HEADER NAMES hiredis HINTS ${HIREDIS_DIR} PATH_SUFFIXES include)
    find_library(HIREDIS_LIB NAMES hiredis HINTS ${HIREDIS_DIR} PATH_SUFFIXES lib)
  else()
    find_package(hiredis REQUIRED)
    find_path(HIREDIS_HEADER hiredis)
    find_library(HIREDIS_LIB hiredis)
  endif()
  message(STATUS "Hiredis library is ${HIREDIS_LIB}")
  message(STATUS "Hiredis headers are ${HIREDIS_HEADER}")
  list(APPEND AMS_APP_INCLUDES ${HIREDIS_HEADER})
  list(APPEND AMS_APP_LIBRARIES ${HIREDIS_LIB})
  list(APPEND AMS_APP_DEFINES "-D__ENABLE_REDIS__")

  if (REDIS_PLUS_PLUS_DIR)
    find_path(REDIS_PLUS_PLUS_HEADER NAMES sw PATHS ${REDIS_PLUS_PLUS_DIR} PATH_SUFFIXES include)
    find_library(REDIS_PLUS_PLUS_LIB NAMES redis++ PATHS ${REDIS_PLUS_PLUS_DIR} PATH_SUFFIXES lib)
  else()
    find_path(REDIS_PLUS_PLUS_HEADER sw)
    find_library(REDIS_PLUS_PLUS_LIB redis++)
  endif()
  message(STATUS "Redis++ library is ${REDIS_PLUS_PLUS_LIB}")
  list(APPEND AMS_APP_INCLUDES ${REDIS_PLUS_PLUS_HEADER})
  list(APPEND AMS_APP_LIBRARIES ${REDIS_PLUS_PLUS_LIB})
endif() # WITH_REDIS

if (WITH_HDF5)
  find_package(HDF5 NAMES hdf5 COMPONENTS C shared REQUIRED NO_DEFAULT_PATH PATHS ${HDF5_Dir} ${HDF5_Dir}/share/cmake)
  list(APPEND AMS_APP_INCLUDES ${HDF5_INCLUDE_DIR})
  list(APPEND AMS_APP_LIBRARIES ${HDF5_C_SHARED_LIBRARY})
  list(APPEND AMS_APP_DEFINES "-D__ENABLE_HDF5__")
  message(STATUS "HDF5 Shared Library: ${HDF5_C_SHARED_LIBRARY}")
  message(STATUS "HDF5 Include directories: ${HDF5_INCLUDE_DIR}")
endif() # WITH_HDF5

if (WITH_RMQ)
  if (WITH_CUDA)
    add_compile_definitions(THRUST_IGNORE_CUB_VERSION_CHECK)
  endif()
  list(APPEND AMS_APP_DEFINES "-D__ENABLE_RMQ__")

  find_package(amqpcpp REQUIRED)
  get_target_property(amqpcpp_INCLUDE_DIR amqpcpp INTERFACE_INCLUDE_DIRECTORIES)
  list(APPEND AMS_APP_INCLUDES ${amqpcpp_INCLUDE_DIR})

  find_package(OpenSSL REQUIRED)
  if (OPENSSL_FOUND)
    list(APPEND AMS_APP_INCLUDES ${OPENSSL_INCLUDE_DIR})
    list(APPEND AMS_APP_LIBRARIES "${OPENSSL_LIBRARIES}")
    list(APPEND AMS_APP_LIBRARIES ssl)
    message(STATUS "OpenSSL includes found: " ${OPENSSL_INCLUDE_DIR})
    message(STATUS "OpenSSL libraries found: " ${OPENSSL_LIBRARIES})
  else()
    message(STATUS "OpenSSL Not Found")
  endif()

  find_package(libevent REQUIRED) # event loop library
  list(APPEND AMS_APP_INCLUDES ${LIBEVENT_INCLUDE_DIR})
  list(APPEND AMS_APP_LIBRARIES "${LIBEVENT_LIBRARIES}")
  list(APPEND AMS_APP_LIBRARIES amqpcpp event_pthreads event)
endif() # WITH_RMQ

# ------------------------------------------------------------------------------
if(NOT DEFINED UMPIRE_DIR)
  message(FATAL_ERROR "Missing required 'UMPIRE_DIR' variable pointing to an installed Umpire ${UMPIRE_DIR}")
endif()

find_package(UMPIRE REQUIRED
             NO_DEFAULT_PATH
             PATHS ${UMPIRE_DIR}/share/umpire/cmake ${UMPIRE_DIR}/lib/cmake/umpire)
list(APPEND AMS_APP_LIBRARIES umpire)
list(APPEND AMS_APP_INCLUDES ${UMPIRE_INCLUDE_DIR})

# ------------------------------------------------------------------------------
find_package(Threads REQUIRED)

# ------------------------------------------------------------------------------
if (WITH_TORCH)
  find_package(Torch REQUIRED)
  # This is annoying, torch populates all my cuda flags
  # and resets them
  set(CMAKE_CUDA_FLAGS "")
  set(CMAKE_CUDA_ARCHITECTURES ON)

  list(APPEND AMS_APP_INCLUDES "${TORCH_INCLUDE_DIRS}")
  list(APPEND AMS_APP_LIBRARIES "${TORCH_LIBRARIES}")

  list(APPEND AMS_APP_DEFINES "-D__ENABLE_TORCH__")
endif()

# ------------------------------------------------------------------------------
if (WITH_FAISS)
  ## TODO: still need to create FindFaiss.cmake
  #find_package(FAISS REQUIRED HINTS ${FAISS_DIR})
  #include(${PROJECT_SOURCE_DIR}/cmake/FindFAISS.cmake)

  if (FAISS_DIR)
    message(STATUS "FAISS_DIR = ${FAISS_DIR}")
    set(FAISS_INCLUDE_DIRS "${FAISS_DIR}/include")
    set(FAISS_LIB_DIR "${FAISS_DIR}/lib")
    set(FAISS_LIB64_DIR "${FAISS_DIR}/lib64")
    set(FAISS_LIBRARIES "faiss")
  else()
    message(FATAL_ERROR "Cannot find FAISS. FAISS_DIR is not defined.")
  endif()

  ## TODO: we need to make a FindFaiss.cmake
  list(APPEND AMS_APP_INCLUDES "${FAISS_INCLUDE_DIRS}")
  list(APPEND AMS_APP_LIB_DIRS "${FAISS_LIB_DIR}" "${FAISS_LIB64_DIR}")
  list(APPEND AMS_APP_LIBRARIES "${FAISS_LIBRARIES}")
  list(APPEND AMS_APP_DEFINES "-D__ENABLE_FAISS__")


  find_package(OpenMP)
  if (OPENMP_FOUND)
     set (CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${OpenMP_C_FLAGS}")
     set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
     set (CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} ${OpenMP_EXE_LINKER_FLAGS}")
  endif()
endif()

if (WITH_RZ)
  find_package(MPI REQUIRED)
  add_subdirectory(rz)
  list(APPEND AMS_APP_INCLUDES "${RZ_AMS_INCLUDES}" "${MPI_INCLUDE_PATH}")
  list(APPEND AMS_APP_LIB_DIRS "${RZ_AMS_LIBDIRS}")
  list(APPEND AMS_APP_LIBRARIES "${RZ_AMS_LIBRARIES}" "${MPI_C_LIBRARIES}")
  list(APPEND AMS_APP_DEFINES "${RZ_AMS_DEFINES}")
endif()

if (WITH_PERFFLOWASPECT)
  find_package(perfflowaspect CONFIG REQUIRED)

  list(APPEND AMS_APP_DEFINES "-D__ENABLE_PERFFLOWASPECT__")
  list(APPEND AMS_APP_LIB_DIRS "${PERFFLOWASPECT_LIB_DIR}")

  list(APPEND AMS_APP_LIBRARIES "perfflow_runtime")
endif()

# TODO: currently we are focusing only on BLASS. Try out other libraries.
set(BLA_VENDER OpenBLAS)
find_package(BLAS REQUIRED)
list(APPEND AMS_APP_LIBRARIES "${BLAS_LIBRARIES}")

add_subdirectory(src)

# ------------------------------------------------------------------------------
if (WITH_EXAMPLES)
  #The AMS Runtime does not depend on MFEM. The
  #examples we use do.
  if (WITH_MPI)
    list(APPEND AMS_EXAMPLE_LIBRARIES MPI::MPI_CXX)
    list(APPEND AMS_EXAMPLE_DEFINES "-D__ENABLE_MPI__")
  endif()

  list(APPEND AMS_EXAMPLE_LIBRARIES umpire)
  list(APPEND AMS_EXAMPLE_INCLUDES ${UMPIRE_INCLUDE_DIR})

  if (MFEM_DIR)
    include(${PROJECT_SOURCE_DIR}/cmake/FindMFEM.cmake)
  else()
    find_package(MFEM REQUIRED)
  endif()
  list(APPEND AMS_EXAMPLE_INCLUDES "${MFEM_INCLUDE_DIRS}")
  list(APPEND AMS_EXAMPLE_LIBRARIES "${MFEM_LIBRARIES}")
  list(APPEND AMS_EXAMPLE_LIB_DIRS "${MFEM_LIB_DIR}")

  if (WITH_REDIS)
    list(APPEND AMS_EXAMPLE_DEFINES "-D__ENABLE_REDIS__")
  endif()

  if (WITH_RMQ)
    list(APPEND AMS_EXAMPLE_DEFINES "-D__ENABLE_RMQ__")
    list(APPEND AMS_EXAMPLE_INCLUDES ${amqpcpp_INCLUDE_DIR})
    list(APPEND AMS_EXAMPLE_INCLUDES ${OPENSSL_INCLUDE_DIR})
    list(APPEND AMS_EXAMPLE_INCLUDES ${LIBEVENT_INCLUDE_DIR})
    list(APPEND AMS_EXAMPLE_LIBRARIES OpenSSL::SSL amqpcpp pthread event_pthreads event)
    if (WITH_MPI)
      list(APPEND AMS_EXAMPLE_LIBRARIES MPI::MPI_CXX)
    endif()
  endif()

  if (WITH_CUDA)
    list(APPEND AMS_EXAMPLE_LIBRARIES ${CUDA_LIBRARIES} cuda)
    list(APPEND AMS_EXAMPLE_DEFINES "-D__ENABLE_CUDA__")
  endif()

  if (WITH_CALIPER)
    list(APPEND AMS_EXAMPLE_INCLUDES ${caliper_INCLUDE_DIR})
    list(APPEND AMS_EXAMPLE_LIBRARIES caliper)
    list(APPEND AMS_EXAMPLE_DEFINES "-D__ENABLE_CALIPER__")
  endif()

  if (WITH_FAISS)
    list(APPEND AMS_EXAMPLE_DEFINES "-D__ENABLE_FAISS__")
  endif()

  if (WITH_TORCH)
    list(APPEND AMS_EXAMPLE_DEFINES "-D__ENABLE_TORCH__")
  endif()

  if (WITH_PERFFLOWASPECT)
    list(APPEND AMS_EXAMPLE_DEFINES "-D__ENABLE_PERFFLOWASPECT__")
  endif()

  add_subdirectory(examples)
endif()

if (WITH_TESTS)
  include(CTest)
  add_subdirectory(tests)
endif()

# ------------------------------------------------------------------------------
