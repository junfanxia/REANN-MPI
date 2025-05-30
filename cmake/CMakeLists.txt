########################################
# CMake build system
# This file is part of LAMMPS
# Created by Christoph Junghans and Richard Berger
cmake_minimum_required(VERSION 3.10)
# set policy to silence warnings about ignoring <PackageName>_ROOT but use it
if(POLICY CMP0074)
  cmake_policy(SET CMP0074 NEW)
endif()

########################################

project(lammps CXX)
set(SOVERSION 0)

get_filename_component(LAMMPS_DIR ${CMAKE_CURRENT_SOURCE_DIR}/.. ABSOLUTE)
get_filename_component(LAMMPS_LIB_BINARY_DIR ${CMAKE_BINARY_DIR}/lib ABSOLUTE)

#############################################################
add_definitions(-D MODE_EINSUM=true)
if(BUILD_FLOAT)
  add_definitions(-D MODE_FLOAT=true)
else()
  add_definitions(-D MODE_FLOAT=false)
endif()

SET(CMAKE_CXX_FLAGS_RELEASE "$ENV{CXXFLAGS} -Ofast -DNDEBUG")
if(BUILD_CUDA)
  SET(CMAKE_PREFIX_PATH     ${LAMMPS_DIR}/pkg/libtorch-1.12.1-gpu)   # add to link the libtorch-gpu
  set(CUDA_TOOLKIT_ROOT_DIR "/usr/local/cuda-12.3/")
  #set(CMAKE_CUDA_COMPILER "/usr/local/cuda-11.7/bin/")
  set(CUDNN_LIBRARY "/usr/local/cuda-12.3/lib64/")
  set(CUDNN_INCLUDE_DIR "/usr/local/cuda-12.3/include/")
  message(STATUS "Building with libtorch ON GPU
    -- libtorch        :    ${CMAKE_PREFIX_PATH}
    -- CUDA  ROOT      :    ${CUDA_TOOLKIT_ROOT_DIR}
    -- CUDNN libraries :    ${CUDNN_LIBRARY}
    -- CUDNN includes  :    ${CUDNN_INCLUDE_DIR};
  ")
  add_definitions(-D MODE_CUDA=true)
else()
  SET(CMAKE_PREFIX_PATH     ${LAMMPS_DIR}/pkg/libtorch-1.12.1-cpu)   # add to link the libtorch-cpu
  message(STATUS "Building with libtorch ON CPU
    -- Libtorch        :    ${CMAKE_PREFIX_PATH}
  ")
  add_definitions(-D MODE_CUDA=false)
endif()

########################################
set(LAMMPS_SOURCE_DIR     ${LAMMPS_DIR}/src)
set(LAMMPS_LIB_SOURCE_DIR ${LAMMPS_DIR}/lib)
set(LAMMPS_DOC_DIR        ${LAMMPS_DIR}/doc)
set(LAMMPS_TOOLS_DIR      ${LAMMPS_DIR}/tools)
set(LAMMPS_PYTHON_DIR     ${LAMMPS_DIR}/python)
set(LAMMPS_POTENTIALS_DIR ${LAMMPS_DIR}/potentials)


find_package(Git)
find_package(Torch REQUIRED)   # add to find the torch in the path

# by default, install into $HOME/.local (not /usr/local), so that no root access (and sudo!!) is needed
if(CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
  set(CMAKE_INSTALL_PREFIX "$ENV{HOME}/.local" CACHE PATH "Default install path" FORCE)
endif()

# If enabled, no need to use LD_LIBRARY_PATH / DYLD_LIBRARY_PATH when installed
option(LAMMPS_INSTALL_RPATH "Set runtime path for shared libraries linked to LAMMPS binaries" OFF)
if (LAMMPS_INSTALL_RPATH)
  set(CMAKE_INSTALL_RPATH ${CMAKE_INSTALL_FULL_LIBDIR})
  set(CMAKE_INSTALL_RPATH_USE_LINK_PATH ON)
endif()

# Cmake modules/macros are in a subdirectory to keep this file cleaner
set(CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/Modules)

# make sure LIBRARY_PATH is set if environment variable is set
if(DEFINED ENV{LIBRARY_PATH})
  list(APPEND CMAKE_LIBRARY_PATH "$ENV{LIBRARY_PATH}")
  message(STATUS "Appending $ENV{LIBRARY_PATH} to CMAKE_LIBRARY_PATH: ${CMAKE_LIBRARY_PATH}")
endif()

include(LAMMPSUtils)

get_lammps_version(${LAMMPS_SOURCE_DIR}/version.h PROJECT_VERSION)

include(PreventInSourceBuilds)

if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CXX_FLAGS)
  set(CMAKE_BUILD_TYPE RelWithDebInfo CACHE STRING "Choose the type of build, options are: None Debug Release RelWithDebInfo MinSizeRel." FORCE)
endif(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CXX_FLAGS)
string(TOUPPER "${CMAKE_BUILD_TYPE}" BTYPE)

# check for files auto-generated by make-based buildsystem
# this is fast, so check for it all the time
check_for_autogen_files(${LAMMPS_SOURCE_DIR})

######################################################################
# compiler tests
# these need ot be done early (before further tests).
#####################################################################
include(CheckIncludeFileCXX)

# set required compiler flags and compiler/CPU arch specific optimizations
if(${CMAKE_CXX_COMPILER_ID} STREQUAL "Intel")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -restrict")
  if(CMAKE_CXX_COMPILER_VERSION VERSION_EQUAL 17.3 OR CMAKE_CXX_COMPILER_VERSION VERSION_EQUAL 17.4)
    set(CMAKE_TUNE_DEFAULT "-xCOMMON-AVX512")
  else()
    set(CMAKE_TUNE_DEFAULT "-xHost")
  endif()
endif()

# we require C++11 without extensions
set(CMAKE_CXX_STANDARD 17) #set(CMAKE_CXX_STANDARD 11)   # modify  for c++14 is used for libtorch
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF CACHE BOOL "Use compiler extensions")

########################################################################
# User input options                                                   #
########################################################################
set(LAMMPS_MACHINE "" CACHE STRING "Suffix to append to lmp binary (WON'T enable any features automatically")
mark_as_advanced(LAMMPS_MACHINE)
if(LAMMPS_MACHINE)
  set(LAMMPS_MACHINE "_${LAMMPS_MACHINE}")
endif()
set(LAMMPS_BINARY lmp${LAMMPS_MACHINE})

option(BUILD_SHARED_LIBS "Build shared library" OFF)
if(BUILD_SHARED_LIBS) # for all pkg libs, mpi_stubs and linalg
  set(CMAKE_POSITION_INDEPENDENT_CODE ON)
endif()

option(BUILD_TOOLS "Build and install LAMMPS tools (msi2lmp, binary2txt, chain)" OFF)
option(BUILD_LAMMPS_SHELL "Build and install the LAMMPS shell" OFF)

include(GNUInstallDirs)
file(GLOB ALL_SOURCES ${LAMMPS_SOURCE_DIR}/[^.]*.cpp)
file(GLOB MAIN_SOURCES ${LAMMPS_SOURCE_DIR}/main.cpp)
list(REMOVE_ITEM ALL_SOURCES ${MAIN_SOURCES})
add_library(lammps ${ALL_SOURCES})
add_executable(lmp ${MAIN_SOURCES})

target_link_libraries(lammps PRIVATE "${TORCH_LIBRARIES}")   # add to link the library
target_link_libraries(lmp PRIVATE lammps)
set_target_properties(lmp PROPERTIES OUTPUT_NAME ${LAMMPS_BINARY})


install(TARGETS lmp EXPORT LAMMPS_Targets DESTINATION ${CMAKE_INSTALL_BINDIR})


option(CMAKE_VERBOSE_MAKEFILE "Generate verbose Makefiles" OFF)

set(STANDARD_PACKAGES ASPHERE BODY CLASS2 COLLOID COMPRESS DIPOLE
  GRANULAR KSPACE LATTE MANYBODY MC MESSAGE MISC MLIAP MOLECULE PERI POEMS
  QEQ REPLICA RIGID SHOCK SPIN SNAP SRD KIM PYTHON MSCG MPIIO VORONOI
  USER-ADIOS USER-ATC USER-AWPMD USER-BOCS USER-CGDNA USER-MESODPD USER-CGSDK
  USER-COLVARS USER-DIFFRACTION USER-DPD USER-DRUDE USER-EFF USER-FEP USER-H5MD
  USER-LB USER-MANIFOLD USER-MEAMC USER-MESONT USER-MGPT USER-MISC USER-MOFFF
  USER-MOLFILE USER-NETCDF USER-PHONON USER-PLUMED USER-PTM USER-QTB
  USER-REACTION USER-REAXC USER-SCAFACOS USER-SDPD USER-SMD USER-SMTBQ USER-SPH
  USER-TALLY USER-UEF USER-VTK USER-QUIP USER-QMMM USER-YAFF)

set(SUFFIX_PACKAGES CORESHELL GPU KOKKOS OPT USER-INTEL USER-OMP)

foreach(PKG ${STANDARD_PACKAGES} ${SUFFIX_PACKAGES})
  option(PKG_${PKG} "Build ${PKG} Package" OFF)
endforeach()

######################################################
# packages with special compiler needs or external libs
######################################################
target_include_directories(lammps PUBLIC $<BUILD_INTERFACE:${LAMMPS_SOURCE_DIR}>)

if(PKG_USER-ADIOS)
  # The search for ADIOS2 must come before MPI because
  # it includes its own MPI search with the latest FindMPI.cmake
  # script that defines the MPI::MPI_C target
  enable_language(C)
  find_package(ADIOS2 REQUIRED)
  target_link_libraries(lammps PRIVATE adios2::adios2)
endif()

if(NOT CMAKE_CROSSCOMPILING)
  set(MPI_CXX_SKIP_MPICXX TRUE)
  find_package(MPI QUIET)
  option(BUILD_MPI "Build MPI version" ${MPI_FOUND})
else()
  option(BUILD_MPI "Build MPI version" OFF)
endif()

if(BUILD_MPI)
  # We use a non-standard procedure to cross-compile with MPI on Windows
  if((CMAKE_SYSTEM_NAME STREQUAL Windows) AND CMAKE_CROSSCOMPILING)
    include(MPI4WIN)
    target_link_libraries(lammps PUBLIC MPI::MPI_CXX)
  else()
    find_package(MPI REQUIRED)
    target_link_libraries(lammps PUBLIC MPI::MPI_CXX)
    option(LAMMPS_LONGLONG_TO_LONG "Workaround if your system or MPI version does not recognize 'long long' data types" OFF)
    if(LAMMPS_LONGLONG_TO_LONG)
      target_compile_definitions(lammps PRIVATE -DLAMMPS_LONGLONG_TO_LONG)
    endif()
  endif()
else()
  file(GLOB MPI_SOURCES ${LAMMPS_SOURCE_DIR}/STUBS/mpi.cpp)
  add_library(mpi_stubs STATIC ${MPI_SOURCES})
  set_target_properties(mpi_stubs PROPERTIES OUTPUT_NAME lammps_mpi_stubs${LAMMPS_MACHINE})
  target_include_directories(mpi_stubs PUBLIC $<BUILD_INTERFACE:${LAMMPS_SOURCE_DIR}/STUBS>)
  if(BUILD_SHARED_LIBS)
    target_link_libraries(lammps PRIVATE mpi_stubs)
    target_include_directories(lammps INTERFACE $<BUILD_INTERFACE:${LAMMPS_SOURCE_DIR}/STUBS>)
    target_compile_definitions(lammps INTERFACE $<INSTALL_INTERFACE:LAMMPS_LIB_NO_MPI>)
  else()
    target_link_libraries(lammps PUBLIC mpi_stubs)
  endif()
  add_library(MPI::MPI_CXX ALIAS mpi_stubs)
endif()

set(LAMMPS_SIZES "smallbig" CACHE STRING "LAMMPS integer sizes (smallsmall: all 32-bit, smallbig: 64-bit #atoms #timesteps, bigbig: also 64-bit imageint, 64-bit atom ids)")
set(LAMMPS_SIZES_VALUES smallbig bigbig smallsmall)
set_property(CACHE LAMMPS_SIZES PROPERTY STRINGS ${LAMMPS_SIZES_VALUES})
validate_option(LAMMPS_SIZES LAMMPS_SIZES_VALUES)
string(TOUPPER ${LAMMPS_SIZES} LAMMPS_SIZES)
target_compile_definitions(lammps PUBLIC -DLAMMPS_${LAMMPS_SIZES})

# posix_memalign is not available on Windows
if(${CMAKE_SYSTEM_NAME} STREQUAL "Windows")
  set(LAMMPS_MEMALIGN "0" CACHE STRING "posix_memalign() is not available on Windows" FORCE)
else()
  set(LAMMPS_MEMALIGN "64" CACHE STRING "enables the use of the posix_memalign() call instead of malloc() when large chunks or memory are allocated by LAMMPS. Set to 0 to disable")
endif()
if(NOT ${LAMMPS_MEMALIGN} STREQUAL "0")
  target_compile_definitions(lammps PRIVATE -DLAMMPS_MEMALIGN=${LAMMPS_MEMALIGN})
endif()

option(LAMMPS_EXCEPTIONS "enable the use of C++ exceptions for error messages (useful for library interface)" ${ENABLE_TESTING})
if(LAMMPS_EXCEPTIONS)
  target_compile_definitions(lammps PUBLIC -DLAMMPS_EXCEPTIONS)
endif()

# "hard" dependencies between packages resulting
# in an error instead of skipping over files
pkg_depends(MLIAP SNAP)
pkg_depends(MPIIO MPI)
pkg_depends(USER-ATC MANYBODY)
pkg_depends(USER-LB MPI)
pkg_depends(USER-PHONON KSPACE)
pkg_depends(USER-SCAFACOS MPI)

# detect if we may enable OpenMP support by default
set(BUILD_OMP_DEFAULT OFF)
find_package(OpenMP QUIET)
if(OpenMP_FOUND)
  check_include_file_cxx(omp.h HAVE_OMP_H_INCLUDE)
  if(HAVE_OMP_H_INCLUDE)
    set(BUILD_OMP_DEFAULT ON)
  endif()
endif()

option(BUILD_OMP "Build with OpenMP support" ${BUILD_OMP_DEFAULT})

if(BUILD_OMP)
  find_package(OpenMP REQUIRED)
  check_include_file_cxx(omp.h HAVE_OMP_H_INCLUDE)
  if(NOT HAVE_OMP_H_INCLUDE)
    message(FATAL_ERROR "Cannot find the 'omp.h' header file required for full OpenMP support")
  endif()

  if (((CMAKE_CXX_COMPILER_ID STREQUAL "GNU") AND (CMAKE_CXX_COMPILER_VERSION VERSION_GREATER_EQUAL 9.0)) OR
      (CMAKE_CXX_COMPILER_ID STREQUAL "PGI") OR
      ((CMAKE_CXX_COMPILER_ID STREQUAL "Clang") AND (CMAKE_CXX_COMPILER_VERSION VERSION_GREATER_EQUAL 10.0)) OR
      ((CMAKE_CXX_COMPILER_ID STREQUAL "Intel") AND (CMAKE_CXX_COMPILER_VERSION VERSION_GREATER_EQUAL 19.0)))
    # GCC 9.x and later plus Clang 10.x and later implement strict OpenMP 4.0 semantics for consts.
    # Intel 18.0 was tested to support both, so we switch to OpenMP 4+ from 19.x onward to be safe.
    target_compile_definitions(lammps PRIVATE -DLAMMPS_OMP_COMPAT=4)
  else()
    target_compile_definitions(lammps PRIVATE -DLAMMPS_OMP_COMPAT=3)
  endif()
  target_link_libraries(lammps PRIVATE OpenMP::OpenMP_CXX)
endif()

# Compiler specific features for testing
if(${CMAKE_CXX_COMPILER_ID} STREQUAL "GNU")
  option(ENABLE_COVERAGE "Enable collecting code coverage data" OFF)
  mark_as_advanced(ENABLE_COVERAGE)
  if(ENABLE_COVERAGE)
    if(CMAKE_VERSION VERSION_LESS 3.13)
      if(CMAKE_CXX_FLAGS)
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} --coverage") 
      else()
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_${CMAKE_BUILD_TYPE}_FLAGS} --coverage") 
      endif()
    else()
      target_compile_options(lammps PUBLIC --coverage)
      target_link_options(lammps PUBLIC --coverage)
    endif()
  endif()
endif()

#######################################
# add custom target for IWYU analysis
#######################################
set(ENABLE_IWYU OFF CACHE BOOL "Add 'iwyu' build target to call the include-what-you-use tool")
mark_as_advanced(ENABLE_IWYU)
if(ENABLE_IWYU)
  find_program(IWYU_EXE NAMES include-what-you-use iwyu)
  find_program(IWYU_TOOL NAMES iwyu_tool iwyu-tool iwyu_tool.py)
  if (IWYU_EXE AND IWYU_TOOL)
    add_custom_target(
      iwyu
      ${IWYU_TOOL} -o clang -p ${CMAKE_CURRENT_BINARY_DIR} -- -Xiwyu --mapping_file=${CMAKE_CURRENT_SOURCE_DIR}/iwyu/iwyu-extra-map.imp
      COMMENT "Running IWYU")
    add_dependencies(iwyu lammps)
  else()
    message(FATAL_ERROR "To use IWYU you need the include-what-you-use/iwyu executable"
      "and the iwyu-tool/iwyu_tool script installed in your PATH")
  endif()
endif()

set(ENABLE_SANITIZER "none" CACHE STRING "Select a code sanitizer option (none (default), address, leak, thread, undefined)")
mark_as_advanced(ENABLE_SANITIZER)
set(ENABLE_SANITIZER_VALUES none address leak thread undefined)
set_property(CACHE ENABLE_SANITIZER PROPERTY STRINGS ${ENABLE_SANITIZER_VALUES})
validate_option(ENABLE_SANITIZER ENABLE_SANITIZER_VALUES)
string(TOLOWER ${ENABLE_SANITIZER} ENABLE_SANITIZER)
if(NOT ENABLE_SANITIZER STREQUAL "none")
  if((${CMAKE_CXX_COMPILER_ID} STREQUAL "GNU") OR (${CMAKE_CXX_COMPILER_ID} STREQUAL "Clang"))
    if(CMAKE_VERSION VERSION_LESS 3.13)
      if(CMAKE_CXX_FLAGS)
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=${ENABLE_SANITIZER}")
      else()
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_${CMAKE_BUILD_TYPE}_FLAGS} -fsanitize=${ENABLE_SANITIZER}")
      endif()
    else()
      target_compile_options(lammps PUBLIC -fsanitize=${ENABLE_SANITIZER})
      target_link_options(lammps PUBLIC -fsanitize=${ENABLE_SANITIZER})
    endif()
  else()
    message(WARNING "ENABLE_SANITIZER option not supported by ${CMAKE_CXX_COMPILER_ID} compilers. Ignoring.")
    set(ENABLE_SANITIZER "none")
  endif()
endif()

if(PKG_MSCG OR PKG_USER-ATC OR PKG_USER-AWPMD OR PKG_USER-QUIP OR PKG_LATTE)
  enable_language(C)
  find_package(LAPACK)
  find_package(BLAS)
  if(NOT LAPACK_FOUND OR NOT BLAS_FOUND)
    include(CheckGeneratorSupport)
    if(NOT CMAKE_GENERATOR_SUPPORT_FORTRAN)
      status(FATAL_ERROR "Cannot build internal linear algebra library as CMake build tool lacks Fortran support")
    endif()
    enable_language(Fortran)
    file(GLOB LAPACK_SOURCES ${LAMMPS_LIB_SOURCE_DIR}/linalg/[^.]*.[fF])
    add_library(linalg STATIC ${LAPACK_SOURCES})
    set_target_properties(linalg PROPERTIES OUTPUT_NAME lammps_linalg${LAMMPS_MACHINE})
    set(BLAS_LIBRARIES "$<TARGET_FILE:linalg>")
    set(LAPACK_LIBRARIES "$<TARGET_FILE:linalg>")
  else()
    list(APPEND LAPACK_LIBRARIES ${BLAS_LIBRARIES})
  endif()
endif()

find_package(JPEG QUIET)
option(WITH_JPEG "Enable JPEG support" ${JPEG_FOUND})
if(WITH_JPEG)
  find_package(JPEG REQUIRED)
  target_compile_definitions(lammps PRIVATE -DLAMMPS_JPEG)
  if(CMAKE_VERSION VERSION_LESS 3.12)
    target_include_directories(lammps PRIVATE ${JPEG_INCLUDE_DIRS})
    target_link_libraries(lammps PRIVATE ${JPEG_LIBRARIES})
  else()
    target_link_libraries(lammps PRIVATE JPEG::JPEG)
  endif()
endif()

find_package(PNG QUIET)
find_package(ZLIB QUIET)
if(PNG_FOUND AND ZLIB_FOUND)
  option(WITH_PNG "Enable PNG support" ON)
else()
  option(WITH_PNG "Enable PNG support" OFF)
endif()
if(WITH_PNG)
  find_package(PNG REQUIRED)
  find_package(ZLIB REQUIRED)
  target_link_libraries(lammps PRIVATE PNG::PNG ZLIB::ZLIB)
  target_compile_definitions(lammps PRIVATE -DLAMMPS_PNG)
endif()

find_program(GZIP_EXECUTABLE gzip)
find_package_handle_standard_args(GZIP REQUIRED_VARS GZIP_EXECUTABLE)
option(WITH_GZIP "Enable GZIP support" ${GZIP_FOUND})
if(WITH_GZIP)
  if(GZIP_FOUND OR ((CMAKE_SYSTEM_NAME STREQUAL Windows) AND CMAKE_CROSSCOMPILING))
    target_compile_definitions(lammps PRIVATE -DLAMMPS_GZIP)
  else()
    message(FATAL_ERROR "gzip executable not found")
  endif()
endif()

find_program(FFMPEG_EXECUTABLE ffmpeg)
find_package_handle_standard_args(FFMPEG REQUIRED_VARS FFMPEG_EXECUTABLE)
option(WITH_FFMPEG "Enable FFMPEG support" ${FFMPEG_FOUND})
if(WITH_FFMPEG)
  if(FFMPEG_FOUND OR ((CMAKE_SYSTEM_NAME STREQUAL Windows) AND CMAKE_CROSSCOMPILING))
    target_compile_definitions(lammps PRIVATE -DLAMMPS_FFMPEG)
  else()
    message(FATAL_ERROR "ffmpeg executable not found")
  endif()
endif()

if(BUILD_SHARED_LIBS)
  set(CONFIGURE_REQUEST_PIC "--with-pic")
  set(CMAKE_REQUEST_PIC "-DCMAKE_POSITION_INDEPENDENT_CODE=${CMAKE_POSITION_INDEPENDENT_CODE}")
  set(CUDA_REQUEST_PIC "-Xcompiler ${CMAKE_SHARED_LIBRARY_CXX_FLAGS}")
else()
  set(CONFIGURE_REQUEST_PIC)
  set(CMAKE_REQUEST_PIC)
  set(CUDA_REQUEST_PIC)
endif()

foreach(PKG_WITH_INCL KSPACE PYTHON MLIAP VORONOI USER-COLVARS USER-MOLFILE USER-NETCDF USER-PLUMED USER-QMMM
        USER-QUIP USER-SCAFACOS USER-SMD USER-VTK KIM LATTE MESSAGE MSCG COMPRESS)
  if(PKG_${PKG_WITH_INCL})
    include(Packages/${PKG_WITH_INCL})
  endif()
endforeach()

# optionally enable building script wrappers using swig
option(WITH_SWIG "Build scripting language wrappers with SWIG" OFF)
if(WITH_SWIG)
  get_filename_component(LAMMPS_SWIG_DIR ${LAMMPS_SOURCE_DIR}/../tools/swig ABSOLUTE)
  add_subdirectory(${LAMMPS_SWIG_DIR} swig)
endif()

set(CMAKE_TUNE_FLAGS "${CMAKE_TUNE_DEFAULT}" CACHE STRING "Compiler and machine specific optimization flags (compilation only)")
separate_arguments(CMAKE_TUNE_FLAGS)
foreach(_FLAG ${CMAKE_TUNE_FLAGS})
  target_compile_options(lammps PRIVATE ${_FLAG})
endforeach()
########################################################################
# Basic system tests (standard libraries, headers, functions, types)   #
########################################################################
foreach(HEADER cmath)
  check_include_file_cxx(${HEADER} FOUND_${HEADER})
  if(NOT FOUND_${HEADER})
    message(FATAL_ERROR "Could not find needed header - ${HEADER}")
  endif(NOT FOUND_${HEADER})
endforeach(HEADER)

set(MATH_LIBRARIES "m" CACHE STRING "math library")
mark_as_advanced( MATH_LIBRARIES )
target_link_libraries(lammps PRIVATE ${MATH_LIBRARIES})

######################################
# Generate Basic Style files
######################################
include(StyleHeaderUtils)
RegisterStyles(${LAMMPS_SOURCE_DIR})

########################################################
# Fetch missing external files and archives for packages
########################################################
foreach(PKG ${STANDARD_PACKAGES} ${SUFFIX_PACKAGES})
  if(PKG_${PKG})
    FetchPotentials(${LAMMPS_SOURCE_DIR}/${PKG} ${LAMMPS_POTENTIALS_DIR})
  endif()
endforeach()

##############################################
# add sources of enabled packages
############################################
foreach(PKG ${STANDARD_PACKAGES})
  set(${PKG}_SOURCES_DIR ${LAMMPS_SOURCE_DIR}/${PKG})

  file(GLOB ${PKG}_SOURCES ${${PKG}_SOURCES_DIR}/[^.]*.cpp)
  file(GLOB ${PKG}_HEADERS ${${PKG}_SOURCES_DIR}/[^.]*.h)

  # check for package files in src directory due to old make system
  DetectBuildSystemConflict(${LAMMPS_SOURCE_DIR} ${${PKG}_SOURCES} ${${PKG}_HEADERS})

  if(PKG_${PKG})
    # detects styles in package and adds them to global list
    RegisterStyles(${${PKG}_SOURCES_DIR})

    target_sources(lammps PRIVATE ${${PKG}_SOURCES})
    target_include_directories(lammps PRIVATE ${${PKG}_SOURCES_DIR})
  endif()

  RegisterPackages(${${PKG}_SOURCES_DIR})
endforeach()

# packages that need defines set
foreach(PKG MPIIO)
  if(PKG_${PKG})
    target_compile_definitions(lammps PRIVATE -DLMP_${PKG})
  endif()
endforeach()

# dedicated check for entire contents of accelerator packages
foreach(PKG ${SUFFIX_PACKAGES})
  set(${PKG}_SOURCES_DIR ${LAMMPS_SOURCE_DIR}/${PKG})

  file(GLOB ${PKG}_SOURCES ${${PKG}_SOURCES_DIR}/[^.]*.cpp)
  file(GLOB ${PKG}_HEADERS ${${PKG}_SOURCES_DIR}/[^.]*.h)

  # check for package files in src directory due to old make system
  DetectBuildSystemConflict(${LAMMPS_SOURCE_DIR} ${${PKG}_SOURCES} ${${PKG}_HEADERS})

  RegisterPackages(${${PKG}_SOURCES_DIR})
endforeach()

##############################################
# add lib sources of (simple) enabled packages
############################################
foreach(SIMPLE_LIB POEMS USER-ATC USER-AWPMD USER-H5MD USER-MESONT)
  if(PKG_${SIMPLE_LIB})
    string(REGEX REPLACE "^USER-" "" PKG_LIB "${SIMPLE_LIB}")
    string(TOLOWER "${PKG_LIB}" PKG_LIB)
    if(PKG_LIB STREQUAL mesont)
      enable_language(Fortran)
      file(GLOB_RECURSE ${PKG_LIB}_SOURCES
        ${LAMMPS_LIB_SOURCE_DIR}/${PKG_LIB}/[^.]*.f90)
    else()
      file(GLOB_RECURSE ${PKG_LIB}_SOURCES
        ${LAMMPS_LIB_SOURCE_DIR}/${PKG_LIB}/[^.]*.c
        ${LAMMPS_LIB_SOURCE_DIR}/${PKG_LIB}/[^.]*.cpp)
    endif()
    add_library(${PKG_LIB} STATIC ${${PKG_LIB}_SOURCES})
    set_target_properties(${PKG_LIB} PROPERTIES OUTPUT_NAME lammps_${PKG_LIB}${LAMMPS_MACHINE})
    target_link_libraries(lammps PRIVATE ${PKG_LIB})
    if(PKG_LIB STREQUAL awpmd)
      target_include_directories(awpmd PUBLIC ${LAMMPS_LIB_SOURCE_DIR}/awpmd/systems/interact ${LAMMPS_LIB_SOURCE_DIR}/awpmd/ivutils/include)
    elseif(PKG_LIB STREQUAL h5md)
      target_include_directories(h5md PUBLIC ${LAMMPS_LIB_SOURCE_DIR}/h5md/include ${HDF5_INCLUDE_DIRS})
    else()
      target_include_directories(${PKG_LIB} PUBLIC ${LAMMPS_LIB_SOURCE_DIR}/${PKG_LIB})
    endif()
  endif()
endforeach()

if(PKG_USER-AWPMD)
  target_link_libraries(awpmd PRIVATE ${LAPACK_LIBRARIES})
endif()

if(PKG_USER-ATC)
  if(LAMMPS_SIZES STREQUAL BIGBIG)
    message(FATAL_ERROR "The USER-ATC Package is not compatible with -DLAMMPS_BIGBIG")
  endif()
  target_link_libraries(atc PRIVATE ${LAPACK_LIBRARIES})
  if(BUILD_MPI)
    target_link_libraries(atc PRIVATE MPI::MPI_CXX)
  else()
    target_link_libraries(atc PRIVATE mpi_stubs)
  endif()
  target_include_directories(atc PRIVATE ${LAMMPS_SOURCE_DIR})
  target_compile_definitions(atc PRIVATE -DLAMMPS_${LAMMPS_SIZES})
endif()

if(PKG_USER-H5MD)
  include(Packages/USER-H5MD)
endif()

######################################################################
# packages which selectively include variants based on enabled styles
# e.g. accelerator packages
######################################################################
foreach(PKG_WITH_INCL CORESHELL QEQ USER-OMP USER-SDPD KOKKOS OPT USER-INTEL GPU)
  if(PKG_${PKG_WITH_INCL})
    include(Packages/${PKG_WITH_INCL})
  endif()
endforeach()

######################################################################
# the windows version of LAMMPS requires a couple extra libraries
# and the MPI library - if use - has to be linked right before those
# and after everything else that is compiled locally
######################################################################
if(${CMAKE_SYSTEM_NAME} STREQUAL "Windows")
  target_link_libraries(lammps PRIVATE -lwsock32 -lpsapi)
endif()

######################################################
# Generate style headers based on global list of
# styles registered during package selection
# Generate packages headers from all packages
######################################################
set(LAMMPS_STYLE_HEADERS_DIR ${CMAKE_CURRENT_BINARY_DIR}/styles)

GenerateStyleHeaders(${LAMMPS_STYLE_HEADERS_DIR})
GeneratePackagesHeaders(${LAMMPS_STYLE_HEADERS_DIR})

target_include_directories(lammps PRIVATE ${LAMMPS_STYLE_HEADERS_DIR})

######################################
# Generate lmpinstalledpkgs.h
######################################
set(temp "#ifndef LMP_INSTALLED_PKGS_H\n#define LMP_INSTALLED_PKGS_H\n")
set(temp "${temp}const char * LAMMPS_NS::LAMMPS::installed_packages[] =  {\n")
set(temp_PKG_LIST ${STANDARD_PACKAGES} ${SUFFIX_PACKAGES})
list(SORT temp_PKG_LIST)
foreach(PKG ${temp_PKG_LIST})
    if(PKG_${PKG})
        set(temp "${temp}  \"${PKG}\",\n")
    endif()
endforeach()
set(temp "${temp}  NULL\n};\n#endif\n\n")
message(STATUS "Generating lmpinstalledpkgs.h...")
file(WRITE "${LAMMPS_STYLE_HEADERS_DIR}/lmpinstalledpkgs.h.tmp" "${temp}" )
execute_process(COMMAND ${CMAKE_COMMAND} -E copy_if_different "${LAMMPS_STYLE_HEADERS_DIR}/lmpinstalledpkgs.h.tmp" "${LAMMPS_STYLE_HEADERS_DIR}/lmpinstalledpkgs.h")

######################################
# Generate lmpgitversion.h
######################################
add_custom_target(gitversion COMMAND ${CMAKE_COMMAND}
  -DLAMMPS_DIR="${LAMMPS_DIR}"
  -DGIT_EXECUTABLE="${GIT_EXECUTABLE}"
  -DGIT_FOUND="${GIT_FOUND}"
  -DLAMMPS_STYLE_HEADERS_DIR="${LAMMPS_STYLE_HEADERS_DIR}"
  -P ${CMAKE_CURRENT_SOURCE_DIR}/Modules/generate_lmpgitversion.cmake)
set_property(DIRECTORY APPEND PROPERTY ADDITIONAL_MAKE_CLEAN_FILES ${LAMMPS_STYLE_HEADERS_DIR}/gitversion.h)
add_dependencies(lammps gitversion)

###########################################
# Actually add executable and lib to build
############################################
get_property(LANGUAGES GLOBAL PROPERTY ENABLED_LANGUAGES)
list (FIND LANGUAGES "Fortran" _index)
if(${_index} GREATER -1)
  target_link_libraries(lammps PRIVATE ${CMAKE_Fortran_IMPLICIT_LINK_LIBRARIES})
endif()
set(LAMMPS_CXX_HEADERS angle.h atom.h bond.h citeme.h comm.h compute.h dihedral.h domain.h error.h fix.h force.h group.h improper.h
  input.h info.h kspace.h lammps.h lattice.h library.h lmppython.h lmptype.h memory.h modify.h neighbor.h neigh_list.h output.h
  pair.h pointers.h region.h timer.h universe.h update.h utils.h variable.h)
if(LAMMPS_EXCEPTIONS)
  list(APPEND LAMMPS_CXX_HEADERS exceptions.h)
endif()

set_target_properties(lammps PROPERTIES OUTPUT_NAME lammps${LAMMPS_MACHINE})
set_target_properties(lammps PROPERTIES SOVERSION ${SOVERSION})
target_include_directories(lammps PUBLIC $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}/lammps>)
file(MAKE_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/includes/lammps)
foreach(_HEADER ${LAMMPS_CXX_HEADERS})
  add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/includes/lammps/${_HEADER} COMMAND ${CMAKE_COMMAND} -E copy_if_different ${LAMMPS_SOURCE_DIR}/${_HEADER} ${CMAKE_CURRENT_BINARY_DIR}/includes/lammps/${_HEADER} DEPENDS ${LAMMPS_SOURCE_DIR}/${_HEADER})
  add_custom_target(${_HEADER} DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/includes/lammps/${_HEADER})
  add_dependencies(lammps ${_HEADER})
  if(BUILD_SHARED_LIBS)
    install(FILES ${LAMMPS_SOURCE_DIR}/${_HEADER} DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/lammps)
  endif()
endforeach()
target_include_directories(lammps INTERFACE $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/includes>)
add_library(LAMMPS::lammps ALIAS lammps)
get_target_property(LAMMPS_DEFINES lammps INTERFACE_COMPILE_DEFINITIONS)
set(LAMMPS_API_DEFINES)
foreach(_DEF ${LAMMPS_DEFINES})
  set(LAMMPS_API_DEFINES "${LAMMPS_API_DEFINES} -D${_DEF}")
endforeach()
if(BUILD_SHARED_LIBS)
  install(TARGETS lammps EXPORT LAMMPS_Targets LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR} ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR})
  configure_file(pkgconfig/liblammps.pc.in ${CMAKE_CURRENT_BINARY_DIR}/liblammps${LAMMPS_MACHINE}.pc @ONLY)
  install(FILES ${CMAKE_CURRENT_BINARY_DIR}/liblammps${LAMMPS_MACHINE}.pc DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig)
  install(EXPORT LAMMPS_Targets FILE LAMMPS_Targets.cmake NAMESPACE LAMMPS:: DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/LAMMPS)
  include(CMakePackageConfigHelpers)
  configure_file(LAMMPSConfig.cmake.in ${CMAKE_CURRENT_BINARY_DIR}/LAMMPSConfig.cmake @ONLY)
  write_basic_package_version_file("LAMMPSConfigVersion.cmake" VERSION ${PROJECT_VERSION} COMPATIBILITY ExactVersion)
  install(FILES "${CMAKE_CURRENT_BINARY_DIR}/LAMMPSConfig.cmake" "${CMAKE_CURRENT_BINARY_DIR}/LAMMPSConfigVersion.cmake" DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/LAMMPS)
endif()
install(FILES ${LAMMPS_DOC_DIR}/lammps.1 DESTINATION ${CMAKE_INSTALL_MANDIR}/man1 RENAME ${LAMMPS_BINARY}.1)

include(Tools)
include(Documentation)

###############################################################################
# Install potential and force field files in data directory
###############################################################################
set(LAMMPS_INSTALL_DATADIR ${CMAKE_INSTALL_FULL_DATADIR}/lammps)
install(DIRECTORY ${LAMMPS_POTENTIALS_DIR} DESTINATION ${LAMMPS_INSTALL_DATADIR})
if(BUILD_TOOLS)
  install(DIRECTORY ${LAMMPS_TOOLS_DIR}/msi2lmp/frc_files DESTINATION ${LAMMPS_INSTALL_DATADIR})
endif()

configure_file(etc/profile.d/lammps.sh.in ${CMAKE_BINARY_DIR}/etc/profile.d/lammps.sh @ONLY)
configure_file(etc/profile.d/lammps.csh.in ${CMAKE_BINARY_DIR}/etc/profile.d/lammps.csh @ONLY)
install(
  FILES ${CMAKE_BINARY_DIR}/etc/profile.d/lammps.sh
        ${CMAKE_BINARY_DIR}/etc/profile.d/lammps.csh
  DESTINATION ${CMAKE_INSTALL_SYSCONFDIR}/profile.d
)

###############################################################################
# Install LAMMPS lib and python module into site-packages folder with
# "install-python" target.  Behaves exactly like "make install-python" for
# conventional build.  Only available, if a shared library is built.
# This is primarily for people that only want to use the Python wrapper.
###############################################################################
if(BUILD_SHARED_LIBS)
  if(CMAKE_VERSION VERSION_LESS 3.12)
    # adjust so we find Python 3 versions before Python 2 on old systems with old CMake
    set(Python_ADDITIONAL_VERSIONS 3.9 3.8 3.7 3.6 3.5)
    find_package(PythonInterp) # Deprecated since version 3.12
    if(PYTHONINTERP_FOUND)
        set(Python_EXECUTABLE ${PYTHON_EXECUTABLE})
    endif()
  else()
    find_package(Python COMPONENTS Interpreter)
  endif()
  if (Python_EXECUTABLE)
    add_custom_target(
      install-python ${CMAKE_COMMAND} -E remove_directory build
      COMMAND ${Python_EXECUTABLE} install.py -v ${LAMMPS_SOURCE_DIR}/version.h
      -p ${LAMMPS_PYTHON_DIR}/lammps
      -l ${CMAKE_BINARY_DIR}/liblammps${LAMMPS_MACHINE}${CMAKE_SHARED_LIBRARY_SUFFIX}
      WORKING_DIRECTORY  ${LAMMPS_PYTHON_DIR}
      COMMENT "Installing LAMMPS Python module")
  else()
    add_custom_target(
      install-python
      ${CMAKE_COMMAND} -E echo "Must have Python installed to install the LAMMPS Python module")
  endif()
else()
  add_custom_target(
    install-python
    ${CMAKE_COMMAND} -E echo "Must build LAMMPS as a shared library to use the Python module")
endif()

###############################################################################
# Add LAMMPS python module to "install" target. This is taylored for building
# LAMMPS for package managers and with different prefix settings.
# This requires either a shared library or that the PYTHON package is included.
###############################################################################
if(BUILD_SHARED_LIBS OR PKG_PYTHON)
  if(CMAKE_VERSION VERSION_LESS 3.12)
    find_package(PythonInterp) # Deprecated since version 3.12
    if(PYTHONINTERP_FOUND)
        set(Python_EXECUTABLE ${PYTHON_EXECUTABLE})
    endif()
  else()
    find_package(Python COMPONENTS Interpreter)
  endif()
  if (Python_EXECUTABLE)
    file(MAKE_DIRECTORY ${CMAKE_BINARY_DIR}/python)
    install(CODE "execute_process(COMMAND ${Python_EXECUTABLE} setup.py build -b ${CMAKE_BINARY_DIR}/python install --prefix=${CMAKE_INSTALL_PREFIX} --root=\$ENV{DESTDIR}/ WORKING_DIRECTORY ${LAMMPS_PYTHON_DIR})")
  endif()
endif()

include(Testing)
include(CodeCoverage)
include(CodingStandard)

get_target_property(DEFINES lammps COMPILE_DEFINITIONS)
include(FeatureSummary)
feature_summary(DESCRIPTION "The following tools and libraries have been found and configured:" WHAT PACKAGES_FOUND)
message(STATUS "<<< Build configuration >>>
   Operating System: ${CMAKE_SYSTEM_NAME}
   Build type:       ${CMAKE_BUILD_TYPE}
   Install path:     ${CMAKE_INSTALL_PREFIX}
   Generator:        ${CMAKE_GENERATOR} using ${CMAKE_MAKE_PROGRAM}")
###############################################################################
# Print package summary
###############################################################################
set(ENABLED_PACKAGES)
foreach(PKG ${STANDARD_PACKAGES} ${SUFFIX_PACKAGES})
  if(PKG_${PKG})
    list(APPEND ENABLED_PACKAGES ${PKG})
  endif()
endforeach()
if(ENABLED_PACKAGES)
  list(SORT ENABLED_PACKAGES)
else()
  set(ENABLED_PACKAGES "<None>")
endif()
message(STATUS "Enabled packages: ${ENABLED_PACKAGES}")

message(STATUS "<<< Compilers and Flags: >>>
-- C++ Compiler:     ${CMAKE_CXX_COMPILER}
      Type:          ${CMAKE_CXX_COMPILER_ID}
      Version:       ${CMAKE_CXX_COMPILER_VERSION}
      C++ Flags:    ${CMAKE_CXX_FLAGS} ${CMAKE_CXX_FLAGS_${BTYPE}}
      Defines:       ${DEFINES}")
get_target_property(OPTIONS lammps COMPILE_OPTIONS)
if(OPTIONS)
  message("      Options:       ${OPTIONS}")
endif()
get_property(LANGUAGES GLOBAL PROPERTY ENABLED_LANGUAGES)
list (FIND LANGUAGES "Fortran" _index)
if(${_index} GREATER -1)
  message(STATUS "Fortran Compiler: ${CMAKE_Fortran_COMPILER}
      Type:          ${CMAKE_Fortran_COMPILER_ID}
      Version:       ${CMAKE_Fortran_COMPILER_VERSION}
      Fortran Flags:${CMAKE_Fortran_FLAGS} ${CMAKE_Fortran_FLAGS_${BTYPE}}")
endif()
list (FIND LANGUAGES "C" _index)
if(${_index} GREATER -1)
  message(STATUS "C compiler:       ${CMAKE_C_COMPILER}
      Type:          ${CMAKE_C_COMPILER_ID}
      Version:       ${CMAKE_C_COMPILER_VERSION}
      C Flags:      ${CMAKE_C_FLAGS} ${CMAKE_C_FLAGS_${BTYPE}}")
endif()
message(STATUS "<<< Linker flags: >>>")
message(STATUS "Executable name:  ${LAMMPS_BINARY}")
if(CMAKE_VERSION VERSION_GREATER_EQUAL 3.13)
  get_target_property(OPTIONS lammps LINK_OPTIONS)
  if(OPTIONS)
    message(STATUS "Linker options:   ${OPTIONS}")
  endif()
endif()
if(CMAKE_EXE_LINKER_FLAGS)
  message(STATUS "Executable linker flags: ${CMAKE_EXE_LINKER_FLAGS}")
endif()
if(BUILD_SHARED_LIBS)
  message(STATUS "Shared library flags:    ${CMAKE_SHARED_LINKER_FLAGS}")
else()
  message(STATUS "Static library flags:    ${CMAKE_STATIC_LINKER_FLAGS}")
endif()
if(BUILD_MPI)
  message(STATUS "<<< MPI flags >>>
-- MPI_defines:      ${MPI_CXX_COMPILE_DEFINITIONS}
-- MPI includes:     ${MPI_CXX_INCLUDE_PATH}
-- MPI libraries:    ${MPI_CXX_LIBRARIES};${MPI_Fortran_LIBRARIES}")
endif()
if(PKG_GPU)
  message(STATUS "<<< GPU package settings >>>
-- GPU API:          ${GPU_API}")
  if(GPU_API STREQUAL "CUDA")
    message(STATUS "GPU default architecture: ${GPU_ARCH}")
  elseif(GPU_API STREQUAL "HIP")
    message(STATUS "HIP platform:     ${HIP_PLATFORM}")
    message(STATUS "HIP architecture: ${HIP_ARCH}")
    if(HIP_USE_DEVICE_SORT)
      message(STATUS "HIP GPU sorting: on")
    else()
      message(STATUS "HIP GPU sorting: off")
    endif()
  endif()
  message(STATUS "GPU precision:    ${GPU_PREC}")
endif()
if(PKG_KOKKOS)
  message(STATUS "Kokkos Arch: ${KOKKOS_ARCH}")
endif()
if(PKG_KSPACE)
  message(STATUS "<<< FFT settings >>>
-- Primary FFT lib:  ${FFT}")
  if(FFT_SINGLE)
    message(STATUS "Using single precision FFTs")
  else()
    message(STATUS "Using double precision FFTs")
  endif()
  if(FFT_FFTW_THREADS OR FFT_MKL_THREADS)
    message(STATUS "Using threaded FFTs")
  else()
    message(STATUS "Using non-threaded FFTs")
  endif()
  if(PKG_KOKKOS)
    if(Kokkos_ENABLE_CUDA)
      if (${FFT} STREQUAL "KISS")
        message(STATUS "Kokkos FFT: KISS")
      else()
        message(STATUS "Kokkos FFT: cuFFT")
      endif()
    else()
      message(STATUS "Kokkos FFT: ${FFT}")
    endif()
  endif()
endif()
if(BUILD_DOC)
  message(STATUS "<<< Building HTML Manual >>>")
endif()
if(BUILD_TOOLS)
  message(STATUS "<<< Building Tools >>>")
endif()
if(BUILD_LAMMPS_SHELL)
  message(STATUS "<<< Building LAMMPS Shell >>>")
endif()
if(ENABLE_TESTING)
  message(STATUS "<<< Building Unit Tests >>>")
  if(ENABLE_COVERAGE)
    message(STATUS "Collecting code coverage data")
  endif()
endif()
