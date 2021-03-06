# ########################################################################
# Copyright 2015 Advanced Micro Devices, Inc.
# Copyright 2015 Vratis, Ltd.
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
# ########################################################################

# We require 2.8.10 because of the added support to download from https URL's
cmake_minimum_required( VERSION 2.8.10 )

if( CMAKE_GENERATOR MATCHES "NMake" )
	option( NMAKE_COMPILE_VERBOSE "Print VERBOSE compile/link msgs to the console" OFF )
	if( NMAKE_COMPILE_VERBOSE )
		set( CMAKE_START_TEMP_FILE "" )
		set( CMAKE_END_TEMP_FILE "" )
		set( CMAKE_VERBOSE_MAKEFILE 1 )
	endif( )
endif( )

# This has to be initialized before the project() command appears
# Set the default of CMAKE_BUILD_TYPE to be release, unless user specifies with -D.  MSVC_IDE does not use CMAKE_BUILD_TYPE
if( NOT MSVC_IDE AND NOT CMAKE_BUILD_TYPE )
  set( CMAKE_BUILD_TYPE Release CACHE STRING "Choose the type of build, options are: None Debug Release RelWithDebInfo MinSizeRel." )
endif()

# Check if cmake supports the new VERSION tag for project() commands
# SuperBuild.clSPARSE becomes the name of the project with a particular version
if( POLICY CMP0048 )
  cmake_policy( SET CMP0048 NEW )

  project( SuperBuild.clSPARSE VERSION 0.10.2.0 )
else( )
  project( SuperBuild.clSPARSE )

  # Define a version for the code
  if( NOT DEFINED SuperBuild.clSPARSE_VERSION_MAJOR )
    set( SuperBuild.clSPARSE_VERSION_MAJOR 0 )
  endif( )

  if( NOT DEFINED SuperBuild.clSPARSE_VERSION_MINOR )
    set( SuperBuild.clSPARSE_VERSION_MINOR 10 )
  endif( )

  if( NOT DEFINED SuperBuild.clSPARSE_VERSION_PATCH )
    set( SuperBuild.clSPARSE_VERSION_PATCH 2 )
  endif( )

  if( NOT DEFINED SuperBuild.clSPARSE_VERSION_TWEAK )
    set( SuperBuild.clSPARSE_VERSION_TWEAK 0 )
  endif( )
endif( )

set( SuperBuild.clSPARSE_VERSION "${SuperBuild.clSPARSE_VERSION_MAJOR}.${SuperBuild.clSPARSE_VERSION_MINOR}.${SuperBuild.clSPARSE_VERSION_PATCH}")

list( APPEND CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake )

if( MSVC_IDE )
  set( BUILD64 ${CMAKE_CL_64} )
  set_property( GLOBAL PROPERTY USE_FOLDERS TRUE )
else( )
  option( BUILD64 "Build a 64-bit product" ON )
endif( )

# Modify the global find property to help us find libraries like Boost in the correct paths for 64-bit
# Essentially, find_library calls will look for /lib64 instead of /lib; works for windows and linux
if( BUILD64 )
  set_property( GLOBAL PROPERTY FIND_LIBRARY_USE_LIB64_PATHS TRUE )
  message( STATUS "64bit build - FIND_LIBRARY_USE_LIB64_PATHS TRUE" )
else( )
  set_property( GLOBAL PROPERTY FIND_LIBRARY_USE_LIB64_PATHS FALSE )
  message( STATUS "32bit build - FIND_LIBRARY_USE_LIB64_PATHS FALSE" )
endif( )

# Various options below on configuring the build, and how to generate the solution files
option( BUILD_clSPARSE "Setup clSPARSE to use all the external dependencies" ON )
#option( BUILD_clBLAS "Install clBLAS as an external dependency" ON )
option( BUILD_MTX "Download sparse matrix files for bench/test" OFF )
option( BUILD_SAMPLES "Build sample program using clSPARSE lib" OFF )
# Query the user for which version of OpenCL they wish to build the library for
set( BUILD_CLVERSION "1.2" CACHE STRING "The version of OpenCL we wish to compile the library against" )
set_property( CACHE BUILD_CLVERSION PROPERTY STRINGS 2.0 1.2 1.1 )


# Comment this out because this does not work yet
#set( clSPARSE.Dependencies )
#set( clSPARSE.Cmake.Args )
#set( clSPARSE.Samples.Cmake.Args )

# Pass OPENCL_ROOT along to subproject if users provides path
if( DEFINED OPENCL_ROOT )
  list( APPEND clSPARSE.Cmake.Args -DOPENCL_ROOT=${OPENCL_ROOT} )
  list( APPEND clSPARSE.Samples.Cmake.Args -DOPENCL_ROOT=${OPENCL_ROOT} )
endif( )

if( DEFINED NMAKE_COMPILE_VERBOSE )
  list( APPEND clSPARSE.Cmake.Args -DNMAKE_COMPILE_VERBOSE=${NMAKE_COMPILE_VERBOSE} )
endif( )

# If the user selects, download, uncompress, and setup clBLAS
if( BUILD_MTX )
  message( STATUS "Setting up Matrix Market data external..." )
  include( ExternalMTX )
endif( )

if( BUILD_clSPARSE OR BUILD_SAMPLES )
  include( ExternalProject )

  # Main project
  if( BUILD_clSPARSE )
    message( STATUS "Setting up clSPARSE external..." )

    if ( BUILD_SAMPLES )
      # If the user elects to build samples, then we must build and install clSPARSE
      # The install location is set as a subdirectory of the SuperBuild
      ExternalProject_Add(
        clSPARSE
        DEPENDS ${clSPARSE.Dependencies}
        SOURCE_DIR ${PROJECT_SOURCE_DIR}/src
        BINARY_DIR clSPARSE-build
        INSTALL_DIR clSPARSE-package
        CMAKE_ARGS ${clSPARSE.Cmake.Args} -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
        -DBUILD_CLVERSION=${BUILD_CLVERSION} -DCMAKE_INSTALL_PREFIX=<INSTALL_DIR>
      )
    else( )
      # If the user does not want to build samples, then build clSPARSE but don't automatically
      # install it
      ExternalProject_Add(
        clSPARSE
        DEPENDS ${clSPARSE.Dependencies}
        SOURCE_DIR ${PROJECT_SOURCE_DIR}/src
        BINARY_DIR clSPARSE-build
        CMAKE_ARGS ${clSPARSE.Cmake.Args} -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
        -DBUILD_CLVERSION=${BUILD_CLVERSION}
        INSTALL_COMMAND ""
      )
    endif( )

  endif( )

  # Our goal with samples is to demonstrate two things
  # 1.  Demonstrate what simple clSPARSE code looks like to set up and execute common functionality
  # 2.  Demonstrate how an external program would find and link clSPARSE
  if ( BUILD_SAMPLES )
    message( STATUS "Setting up sample programs ...")
    ExternalProject_Get_Property( clSPARSE install_dir )

    ExternalProject_Add(
      clSPARSE-samples
      DEPENDS clSPARSE
      SOURCE_DIR ${PROJECT_SOURCE_DIR}/samples
      BINARY_DIR clSPARSE-samples-build
      CMAKE_ARGS ${clSPARSE.Samples.Cmake.Args} -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
      -DBUILD_CLVERSION=${BUILD_CLVERSION} -DBUILD64=${BUILD64} -DCMAKE_PREFIX_PATH=${install_dir}
      EXCLUDE_FROM_ALL 1
      INSTALL_COMMAND ""
    )
  endif( )
endif( )
