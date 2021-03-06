#------------------------------------------------------------------------------#
# Copyright 2019 Kitware, Inc.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
# http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#------------------------------------------------------------------------------#

cmake_minimum_required(VERSION 3.1)
project(Legion)

#------------------------------------------------------------------------------#
# Some boilerplate to setup nice output directories
#------------------------------------------------------------------------------#

#for multilib distros
include(GNUInstallDirs)

list(INSERT CMAKE_MODULE_PATH 0 "${Legion_SOURCE_DIR}/cmake")
if(NOT CMAKE_ARCHIVE_OUTPUT_DIRECTORY)
  set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${Legion_BINARY_DIR}/lib)
endif()
if(NOT CMAKE_LIBRARY_OUTPUT_DIRECTORY)
  set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${Legion_BINARY_DIR}/lib)
endif()
if(NOT CMAKE_RUNTIME_OUTPUT_DIRECTORY)
  set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${Legion_BINARY_DIR}/bin)
endif()

#library api version, bump from time to time
set(SOVERSION 1)

set(INSTALL_SUFFIX "" CACHE STRING "Suffix to add to installed libraries and binaries")

#------------------------------------------------------------------------------#
# Build type and shared vs static library
#------------------------------------------------------------------------------#
if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CXX_FLAGS)
  set(CMAKE_BUILD_TYPE  Debug CACHE STRING "Choose the type of build" FORCE)
endif(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CXX_FLAGS)

if (CMAKE_BUILD_TYPE AND ( CMAKE_BUILD_TYPE STREQUAL "Debug"))
  # define variable for realm_defines.h
  set(DEBUG_REALM ON)
  # define variable for legion_defines.h
  set(DEBUG_LEGION ON)
endif()

set(BUILD_SHARED_LIBS OFF   CACHE BOOL   "Whether or not to build shared libraries instead of static")

#------------------------------------------------------------------------------#
# Architecture
#------------------------------------------------------------------------------#
if(BUILD_MARCH AND BUILD_MCPU)
  message(FATAL_ERROR "BUILD_MARCH and BUILD_MCPU are incompatible")
endif()

# Try -march first. On platforms that don't support it, GCC will issue
# a hard error, so we'll know not to use it.
if(BUILD_MARCH)
  set(INTERNAL_BUILD_MARCH ${BUILD_MARCH})
else()
  set(INTERNAL_BUILD_MARCH "native")
endif()

include(CheckCXXCompilerFlag)
check_cxx_compiler_flag("-march=${INTERNAL_BUILD_MARCH}" COMPILER_SUPPORTS_MARCH)
if(COMPILER_SUPPORTS_MARCH)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -march=${INTERNAL_BUILD_MARCH}")
elseif(BUILD_MARCH)
  message(FATAL_ERROR "The flag -march=${INTERNAL_BUILD_MARCH} is not supported by the compiler")
else()
  unset(INTERNAL_BUILD_MARCH)
endif()

# Try -mcpu. We do this second because it is deprecated on x86, but
# GCC won't issue a hard error, so we can't tell if it worked or not.
if (NOT INTERNAL_BUILD_MARCH)
  if(BUILD_MCPU)
    set(INTERNAL_BUILD_MCPU ${BUILD_MCPU})
  else()
    set(INTERNAL_BUILD_MCPU "native")
  endif()

  check_cxx_compiler_flag("-mcpu=${INTERNAL_BUILD_MCPU}" COMPILER_SUPPORTS_MCPU)
  if(COMPILER_SUPPORTS_MCPU)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -mcpu=${INTERNAL_BUILD_MCPU}")
  elseif(BUILD_MCPU)
    message(FATAL_ERROR "The flag -mcpu=${INTERNAL_BUILD_MCPU} is not supported by the compiler")
  else()
    unset(INTERNAL_BUILD_MCPU)
  endif()
endif()

# Add flags for Power architectures
check_cxx_compiler_flag("-maltivec -Werror" COMPILER_SUPPORTS_MALTIVEC)
if(COMPILER_SUPPORTS_MALTIVEC)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -maltivec")
endif()
check_cxx_compiler_flag("-mabi=altivec -Werror" COMPILER_SUPPORTS_MABI_ALTIVEC)
if(COMPILER_SUPPORTS_MABI_ALTIVEC)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -mabi=altivec")
endif()
check_cxx_compiler_flag("-mvsx -Werror" COMPILER_SUPPORTS_MVSX)
if(COMPILER_SUPPORTS_MVSX)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -mvsx")
endif()

#------------------------------------------------------------------------------#
# Optimizations
#------------------------------------------------------------------------------#

# For now we want the optimization flags to match on both normal make and cmake
# builds so we override the cmake defaults here for release, this changes 
# -O3 to -O2 and removes -DNDEBUG
set(CMAKE_CXX_FLAGS_RELEASE "-O2")

# For debug we can see if it supports the new -Og flag that gcc added recently
check_cxx_compiler_flag("-Og" COMPILER_SUPPORTS_DEBUG_OPTIMIZATIONS)
if(COMPILER_SUPPORTS_DEBUG_OPTIMIZATIONS)
  set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -Og")
endif()

# Legion can't handle certain optimizations. See:
# https://github.com/StanfordLegion/legion/issues/168#issuecomment-244582958
check_cxx_compiler_flag("-fno-strict-aliasing" COMPILER_SUPPORTS_NO_STRICT_ALIASING)
if(COMPILER_SUPPORTS_NO_STRICT_ALIASING)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fno-strict-aliasing")
endif()

#------------------------------------------------------------------------------#
# Minimum log level
#------------------------------------------------------------------------------#
set(Legion_OUTPUT_LEVEL "DEBUG" CACHE STRING "Compile time logging level")
set_property(CACHE Legion_OUTPUT_LEVEL PROPERTY STRINGS SPEW DEBUG INFO PRINT WARNING ERROR FATAL NONE)

# define variable for realm_defines.h
set(COMPILE_TIME_MIN_LEVEL LEVEL_${Legion_OUTPUT_LEVEL})

#------------------------------------------------------------------------------#
# OpenMP
#------------------------------------------------------------------------------#
option(Legion_USE_OpenMP "Use OpenMP" OFF)
if(Legion_USE_OpenMP)
  option(Legion_OpenMP_GOMP_SUPPORT "Enable GOMP support for OpenMP" ON)
  option(Legion_OpenMP_KMP_SUPPORT "Enable KMP support for OpenMP" ON)

  # define variable for realm_defines.h
  set(REALM_USE_OPENMP ON)
  if(Legion_OpenMP_GOMP_SUPPORT)
    set(REALM_OPENMP_GOMP_SUPPORT ON)
  endif()
  if(Legion_OpenMP_KMP_SUPPORT)
    set(REALM_OPENMP_KMP_SUPPORT ON)
  endif()
endif()

#------------------------------------------------------------------------------#
# Python
#------------------------------------------------------------------------------#
option(Legion_USE_Python "Use Python" OFF)
if(Legion_USE_Python)
  # Note: Legion now supports several versions, so no need to be so forceful
  # set(Python_ADDITIONAL_VERSIONS 2.7)
  find_package(PythonInterp REQUIRED) # Needed to invoke setup.py for install
  find_package(PythonLibs REQUIRED)

  # define variable for realm_defines.h
  set(REALM_USE_PYTHON ON)

  string(REPLACE "." ";" PYTHON_VERSION_SPLIT ${PYTHONLIBS_VERSION_STRING})
  list(GET PYTHON_VERSION_SPLIT 0 PYTHON_VERSION_MAJOR)
endif()

set(BINDINGS_DEFAULT_MODULE "" CACHE STRING "module to load by default in Python bindings, if any")

#------------------------------------------------------------------------------#
# CUDA configuration
#------------------------------------------------------------------------------#
option(Legion_USE_CUDA "Enable support for the CUDA runtime" OFF)
if(Legion_USE_CUDA)
  if(NOT BUILD_SHARED_LIBS)
    set(CUDA_USE_STATIC_CUDA_RUNTIME ON)
  else()
    set(CUDA_USE_STATIC_CUDA_RUNTIME OFF)
  endif()
  find_package(CUDA REQUIRED)
  install(FILES ${Legion_SOURCE_DIR}/cmake/FindCUDA.cmake
    DESTINATION ${CMAKE_INSTALL_DATADIR}/Legion/cmake
  )
  install(FILES ${Legion_SOURCE_DIR}/cmake/newcmake/FindCUDA.cmake
    DESTINATION ${CMAKE_INSTALL_DATADIR}/Legion/cmake/newcmake
  )

  # define variable for legion_defines.h
  set(LEGION_USE_CUDA ON)

  # define variable for realm_defines.h
  set(REALM_USE_CUDA ON)
  set(USE_CUDA ON) # for backwards compatibility
  set(REALM_USE_CUDART_HIJACK ON)
endif()

#------------------------------------------------------------------------------#
# network interface configuration
#------------------------------------------------------------------------------#
set(Legion_NETWORKS "" CACHE STRING "Network backend(s) to use")
separate_arguments(Legion_NETWORKS)

# compatibility mode - Legion_USE_GASNet==ON -> Legion_NETWORKS=gasnet1
option(Legion_USE_GASNet "Enable the distributed GASNet backend" OFF)
if(Legion_USE_GASNet)
  list(APPEND Legion_NETWORKS gasnet1)
endif()

#------------------------------------------------------------------------------#
# GASNet configuration
#------------------------------------------------------------------------------#
if("${Legion_NETWORKS}" MATCHES .*gasnet1.*)
  set(GASNet_THREADING par)
  set(GASNet_PREFERRED_CONDUITS aries gemini ibv)
  find_package(GASNet REQUIRED)
  if(NOT GASNet_THREADING STREQUAL "par")
    message(FATAL_ERROR "GASNet threading mode \"${GASNet_THREADING}\" is not currently supported by Legion")
  endif()
  install(FILES ${Legion_SOURCE_DIR}/cmake/FindGASNet.cmake
    DESTINATION ${CMAKE_INSTALL_DATADIR}/Legion/cmake
  )

  # define variable for realm_defines.h
  set(REALM_USE_GASNET1 ON)
  string(TOUPPER ${GASNet_CONDUIT} CONDUIT)
  set(GASNET_CONDUIT_${CONDUIT} ON)
  # conduits other than udp support MPI interop
  if(NOT ${GASNet_CONDUIT} STREQUAL "udp")
    set(Legion_MPI_INTEROP ON)
  endif()
endif()

#------------------------------------------------------------------------------#
# LLVM configuration
#------------------------------------------------------------------------------#
option(Legion_USE_LLVM "Use LLVM JIT operations" OFF)
option(Legion_ALLOW_MISSING_LLVM_LIBS "Allow LLVM libraries to be missing at runtime" OFF)
option(Legion_LINK_LLVM_LIBS "Link LLVM libraries into Realm" ON)
if(Legion_USE_LLVM)
  set(Legion_LLVM_COMPONENTS irreader jit mcjit x86)
  find_package(LLVM REQUIRED COMPONENTS ${Legion_LLVM_COMPONENTS})
  install(FILES ${Legion_SOURCE_DIR}/cmake/FindLLVM.cmake
    DESTINATION ${CMAKE_INSTALL_DATADIR}/Legion/cmake
  )

  # define variables for realm_defines.h
  set(REALM_USE_LLVM ON)
  if(Legion_ALLOW_MISSING_LLVM_LIBS)
    set(REALM_ALLOW_MISSING_LLVM_LIBS ON)
  endif()
  # just want REALM_LLVM_VERSION=<major><minor>
  string(REPLACE "." ";" LLVM_VERSION_SPLIT ${LLVM_VERSION})
  list(GET LLVM_VERSION_SPLIT 0 1 REALM_LLVM_VERSION_LIST)
  string(REPLACE ";" "" REALM_LLVM_VERSION "${REALM_LLVM_VERSION_LIST}")
endif()

#------------------------------------------------------------------------------#
# HDF5 configuration
#------------------------------------------------------------------------------#
option(Legion_USE_HDF5 "Enable support for HDF5" OFF)
if(Legion_USE_HDF5)
  find_package(HDF5 REQUIRED COMPONENTS C)

  # define variable for realm_defines.h
  set(USE_HDF ON)
endif()

#------------------------------------------------------------------------------#
# libdl configuration
#------------------------------------------------------------------------------#
option(Legion_USE_LIBDL "Enable support for libdl" ON)
if(Legion_USE_LIBDL)
  # define variable for realm_defines.h
  set(USE_LIBDL ON)
endif()

#------------------------------------------------------------------------------#
# HWLOC configuration
#------------------------------------------------------------------------------#
option(Legion_USE_HWLOC "Use hwloc for topology awareness" OFF)
if(Legion_USE_HWLOC)
  find_package(HWLOC REQUIRED)
  install(FILES ${Legion_SOURCE_DIR}/cmake/FindHWLOC.cmake
    DESTINATION ${CMAKE_INSTALL_DATADIR}/Legion/cmake
  )
endif()

#------------------------------------------------------------------------------#
# zlib configuration
#------------------------------------------------------------------------------#
option(Legion_USE_ZLIB "Enable support for zlib" ON)
if(Legion_USE_ZLIB)
  find_package(ZLIB REQUIRED)
  # define variable for legion_defines.h
  set(USE_ZLIB ON)
endif()

#------------------------------------------------------------------------------#
# Miscelaneous other options
#------------------------------------------------------------------------------#
set(Legion_MAX_DIM 3 CACHE STRING "Maximum number of dimensions")
set_property(CACHE Legion_MAX_DIM PROPERTY STRINGS 1 2 3 4 5 6 7 8 9)
mark_as_advanced(Legion_MAX_DIM)

# define variable for legion_defines.h
set(LEGION_MAX_DIM ${Legion_MAX_DIM})
set(REALM_MAX_DIM ${Legion_MAX_DIM})

set(Legion_MAX_FIELDS 512 CACHE STRING "Maximum number of fields allocated to a single field space")
set_property(CACHE Legion_MAX_FIELDS PROPERTY STRINGS 32 64 128 256 512 1024)
mark_as_advanced(Legion_MAX_FIELDS)

# define variable for legion_defines.h
set(MAX_FIELDS ${Legion_MAX_FIELDS})

option(Legion_ENABLE_TLS "Enable support for TLS storage of Legion context" OFF)
mark_as_advanced(Legion_ENABLE_TLS)

# define variable for legion_defines.h
set(ENABLE_LEGION_TLS ${Legion_ENABLE_TLS})

option(Legion_WARNINGS_FATAL "Make all runtime warnings fatal" OFF)
set(LEGION_WARNINGS_FATAL ${Legion_WARNINGS_FATAL})

#------------------------------------------------------------------------------#
# Runtime library targets
#------------------------------------------------------------------------------#
add_subdirectory(runtime)

#------------------------------------------------------------------------------#
# Tools
#------------------------------------------------------------------------------#
add_subdirectory(tools)

#------------------------------------------------------------------------------#
# Documentation
#------------------------------------------------------------------------------#
add_subdirectory(doc)

#------------------------------------------------------------------------------#
# configure header
#------------------------------------------------------------------------------#
# Checking for all defines in the CXX Flags

add_definitions(-D__STDC_FORMAT_MACROS)
set (__STDC_FORMAT_MACROS ON)

string(REPLACE " " ";" FLAGS_LIST ${CMAKE_CXX_FLAGS})
FOREACH(FLAG ${FLAGS_LIST})
  string (FIND ${FLAG} "-D" START_STR)
  if (${START_STR} EQUAL "0")
  string(REPLACE "-D" "" NEW_DEFINE ${FLAG})
  set (${NEW_DEFINE} ON)
  endif()
endforeach()


#------------------------------------------------------------------------------#
# Build-tree package generation
#------------------------------------------------------------------------------#
export(EXPORT LegionTargets
  NAMESPACE Legion::
  FILE ${Legion_BINARY_DIR}/LegionTargets.cmake
)
install(EXPORT LegionTargets
  NAMESPACE Legion::
  FILE LegionTargets.cmake
  DESTINATION ${CMAKE_INSTALL_DATADIR}/Legion/cmake
)

configure_file(
  cmake/LegionConfigCommon.cmake.in
  ${Legion_BINARY_DIR}/LegionConfigCommon.cmake
  @ONLY
)
install(FILES ${Legion_BINARY_DIR}/LegionConfigCommon.cmake
  DESTINATION ${CMAKE_INSTALL_DATADIR}/Legion/cmake
)

configure_file(
  cmake/LegionConfig-build.cmake.in
  ${Legion_BINARY_DIR}/LegionConfig.cmake
  @ONLY
)

install(FILES cmake/LegionConfig-install.cmake
  DESTINATION ${CMAKE_INSTALL_DATADIR}/Legion/cmake
  RENAME LegionConfig.cmake
)

#------------------------------------------------------------------------------#
# Install into the CMake pacakge registry if explicitly selected to do so
#------------------------------------------------------------------------------#
set(CMAKE_EXPORT_NO_PACKAGE_REGISTRY ON CACHE INTERNAL "Disable the export(PACKAGE) command.")
option(Legion_USE_PACKAGE_REGISTRY "Register the build tree with the CMake package registry" OFF)
mark_as_advanced(Legion_USE_PACKAGE_REGISTRY)
if(Legion_USE_PACKAGE_REGISTRY)
  set(CMAKE_EXPORT_NO_PACKAGE_REGISTRY OFF)
endif()
export(PACKAGE Legion)

#------------------------------------------------------------------------------#
# Examples and Applications
#------------------------------------------------------------------------------#
option(Legion_BUILD_ALL "Build all Legion bindings, examples, tutorials and tests" OFF)
option(Legion_BUILD_APPS "Build Legion sample applications" OFF)
option(Legion_BUILD_BINDINGS "Build Legion language bindings" OFF)
option(Legion_BUILD_EXAMPLES "Build Legion examples" OFF)
option(Legion_BUILD_TUTORIAL "Build Legion tutorial" OFF)
option(Legion_BUILD_TESTS "Build Legion tests" OFF)
option(Legion_ENABLE_TESTING "Build and copy testing stuff" OFF)
if(Legion_ENABLE_TESTING)
  enable_testing()
  # if Legion_TEST_LAUNCHER is set, split the string into a list
  separate_arguments(Legion_TEST_LAUNCHER)
  separate_arguments(Legion_TEST_ARGS)
endif()
if(Legion_BUILD_ALL OR Legion_BUILD_APPS OR Legion_BUILD_BINDINGS OR Legion_BUILD_EXAMPLES OR Legion_BUILD_TUTORIAL OR Legion_BUILD_TESTS)
  # Make a namespaced alias so in-build examples can use it the same way as if
  # it were imported
  add_library(Legion::Legion ALIAS Legion)
  add_library(Legion::Realm ALIAS Realm)

  if(Legion_USE_CUDA)
    # CUDA has some issues propogating target interface properties so we
    # have to extract them from the target and manually add them to the nvcc
    # flags
    set(PROP $<TARGET_PROPERTY:Legion::Legion,INTERFACE_COMPILE_OPTIONS>)
    set(CUDA_NVCC_FLAGS "${CUDA_NVCC_FLAGS} ${PROP}")
    set(PROP $<TARGET_PROPERTY:Legion::Legion,INTERFACE_COMPILE_DEFINITIONS>)
    set(CUDA_NVCC_FLAGS "${CUDA_NVCC_FLAGS} $<$<BOOL:${PROP}>:-D$<JOIN:${PROP}, -D>>")
    set(PROP $<TARGET_PROPERTY:Legion::Legion,INTERFACE_INCLUDE_DIRECTORIES>)
    set(CUDA_NVCC_FLAGS "${CUDA_NVCC_FLAGS} $<$<BOOL:${PROP}>:-I$<JOIN:${PROP}, -I>>")
  endif()

  if(Legion_BUILD_ALL OR Legion_BUILD_APPS)
    add_subdirectory(apps)
  endif()
  if(Legion_BUILD_ALL OR Legion_BUILD_BINDINGS)
    add_subdirectory(bindings)
  endif()
  if(Legion_BUILD_ALL OR Legion_BUILD_EXAMPLES)
    add_subdirectory(examples)
  endif()
  if(Legion_BUILD_ALL OR Legion_BUILD_TUTORIAL)
    add_subdirectory(tutorial)
  endif()
  if(Legion_BUILD_ALL OR Legion_BUILD_TESTS)
    add_subdirectory(test)
  endif()
endif()

#------------------------------------------------------------------------------#
# Complex support
#------------------------------------------------------------------------------#

if(Legion_BUILD_ALL OR Legion_BUILD_BINDINGS)
  option(Legion_REDOP_COMPLEX "Use reduction operators for complex types" ON)
else()
  option(Legion_REDOP_COMPLEX "Use reduction operators for complex types" OFF)
endif()
if((Legion_BUILD_ALL OR Legion_BUILD_BINDINGS) AND NOT Legion_REDOP_COMPLEX)
  message(FATAL_ERROR "Bindings require Legion_REDOP_COMPLEX to be set")
endif()
if(Legion_REDOP_COMPLEX)
  # define variable for legion_defines.h
  set(LEGION_REDOP_COMPLEX ON)
endif()
mark_as_advanced(Legion_REDOP_COMPLEX)

#------------------------------------------------------------------------------#
# Configure definition headers
#------------------------------------------------------------------------------#

configure_file(${PROJECT_SOURCE_DIR}/cmake/legion_defines.h.in
  ${PROJECT_BINARY_DIR}/runtime/legion_defines.h @ONLY)
install(FILES ${PROJECT_BINARY_DIR}/runtime/legion_defines.h
  DESTINATION  ${CMAKE_INSTALL_INCLUDEDIR})

configure_file(${PROJECT_SOURCE_DIR}/cmake/realm_defines.h.in
  ${PROJECT_BINARY_DIR}/runtime/realm_defines.h @ONLY)
install(FILES ${PROJECT_BINARY_DIR}/runtime/realm_defines.h
  DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})

#------------------------------------------------------------------------------#
# vim: set tabstop=2 shiftwidth=2 expandtab :
#------------------------------------------------------------------------------#
