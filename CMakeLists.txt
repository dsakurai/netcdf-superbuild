cmake_minimum_required(VERSION 3.13)
project(netcdf_superbuild LANGUAGES C CXX Fortran)

include(ExternalProject)

include(ProcessorCount)
# number of cores on this system.
ProcessorCount(num_processors)

#
# Enable fortran
#

# https://cmake.org/Wiki/CMakeForFortranExample
if (Fortran_COMPILER_NAME MATCHES "gfortran.*")
    # gfortran
    set (CMAKE_Fortran_FLAGS_RELEASE "-funroll-all-loops -fno-f2c -O3")
    set (CMAKE_Fortran_FLAGS_DEBUG   "-fno-f2c -O0 -g")
elseif (Fortran_COMPILER_NAME MATCHES "ifort.*")
    # ifort (untested)
    set (CMAKE_Fortran_FLAGS_RELEASE "-f77rtl -O3")
    set (CMAKE_Fortran_FLAGS_DEBUG   "-f77rtl -O0 -g")
elseif (Fortran_COMPILER_NAME MATCHES "g77")
    # g77
    set (CMAKE_Fortran_FLAGS_RELEASE "-funroll-all-loops -fno-f2c -O3 -m32")
    set (CMAKE_Fortran_FLAGS_DEBUG   "-fno-f2c -O0 -g -m32")
else (Fortran_COMPILER_NAME MATCHES "gfortran.*")
    message ("CMAKE_Fortran_COMPILER full path: " ${CMAKE_Fortran_COMPILER})
    message ("Fortran compiler: " ${Fortran_COMPILER_NAME})
    message ("No optimized Fortran compiler flags are known, we just try -O2...")
    set (CMAKE_Fortran_FLAGS_RELEASE "-O2")
    set (CMAKE_Fortran_FLAGS_DEBUG   "-O0 -g")
endif (Fortran_COMPILER_NAME MATCHES "gfortran.*")


find_package(MPI REQUIRED)

if(NOT MPI_C_COMPILER)
    message(FATAL_ERROR "MPI C compiler is not found.")
endif(NOT MPI_C_COMPILER)

if(NOT MPI_CXX_COMPILER)
    message(FATAL_ERROR "MPI C++ compiler is not found.")
endif(NOT MPI_CXX_COMPILER)

if(NOT MPI_Fortran_COMPILER)
    message(FATAL_ERROR "MPI Fortran compiler is not found.")
endif(NOT MPI_Fortran_COMPILER)

# Pass the compilers down;
# Setting the MPI wrapper as the non-MPI compiler in CMake variables
# works on HDF5 and NetCDF.
# (NetCDF documentation says this is their way;
# HDF 5 uses CMake's FindMPI, which respects this scheme.)
set(compilers 
        "-DCMAKE_C_COMPILER=${MPI_C_COMPILER}" # NetCDF just uses the c compiler variable for MPI
        # "-DMPI_C_COMPILER=${MPI_C_COMPILER}"
        "-DCMAKE_CXX_COMPILER=${MPI_CXX_COMPILER}"
        # "-DMPI_CXX_COMPILER=${MPI_CXX_COMPILER}"
        "-DCMAKE_Fortran_COMPILER=${MPI_Fortran_COMPILER}"
        # "-DMPI_Fortran_COMPILER=${MPI_Fortran_COMPILER}"
    )


ExternalProject_Add(hdf5
    SOURCE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/hdf5"

    CMAKE_ARGS
        -DCMAKE_BUILD_TYPE=Release
        "-DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX}"
        ${compilers}

        -DHDF5_ENABLE_PARALLEL=Yes
        -DHDF5_ENABLE_THREADSAFE=No # parallel version means not thread-safe
        -DHDF5_BUILD_CPP_LIB=No
        -DBUILD_SHARED_LIBS=Yes
        -DHDF5_ENABLE_Z_LIB_SUPPORT=Yes
        -DHDF5_ENABLE_SZIP_SUPPORT=Yes
        -DHDF5_ENABLE_SZIP_ENCODING=Yes

    BUILD_COMMAND make "-j${num_processors}"
)

# The HDF5 this script installed.
# We do a sanity-check before configuring the NetCDF (C language).
set(HDF5_DIR "${CMAKE_INSTALL_PREFIX}/share/cmake/hdf5")

ExternalProject_Add(netcdf_c
    SOURCE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/netcdf_c"

    CMAKE_ARGS
        -DCMAKE_BUILD_TYPE=Release
        "-DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX}"
        ${compilers}

        "-DHDF5_DIR=${HDF5_DIR}"

        "-DENABLE_NETCDF_4=ON" 
        "-DENABLE_DAP=OFF"
        "-DBUILD_UTILITIES=ON"
        "-DBUILD_SHARED_LIBS=ON"
        "-DENABLE_PARALLEL4=ON"
        "-DENABLE_PNETCDF=OFF" # disabled because parallel NetCDF 4 handles parallelization
        "-DENABLE_CDF5=OFF"    # we guess CDF 5 is not necessary
        "-DENABLE_TESTS=ON" 
        "-DENABLE_PARALLEL_TESTS=ON"

    BUILD_COMMAND make "-j${num_processors}"
)

# Ensure that NetCDF (C) links to the HDF 5 this project built.
# Before configuring the NetCDF, we check that the CMake config file was generated.
ExternalProject_Add_Step(netcdf_c
    check_hdf5_cmake_config
    DEPENDERS configure
    DEPENDS "${HDF5_DIR}/hdf5-config.cmake"
    )

set(NETCDF_INCLUDE_DIR "${CMAKE_INSTALL_PREFIX}/include")
set(NETCDF_C_LIBRARY     "${CMAKE_INSTALL_PREFIX}/lib64/libnetcdf.so")

ExternalProject_Add(netcdf_fortran
    SOURCE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/netcdf_fortran"

    CMAKE_ARGS
        -DCMAKE_BUILD_TYPE=Release
        "-DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX}"
        ${compilers}

        "-DNETCDF_INCLUDE_DIR=${NETCDF_INCLUDE_DIR}"
        "-DNETCDF_C_LIBRARY=${NETCDF_C_LIBRARY}"

        # NetCDF Fortran automatically follows the configuration
        # NetCDF C 

    BUILD_COMMAND make "-j${num_processors}"
)

# Make sure NetCDF Fortran links to the 
# NetCDF C this project has built
ExternalProject_Add_Step(netcdf_fortran
    check_netcdf_c_lib_exists
    DEPENDERS configure
    DEPENDS "${NETCDF_C_LIBRARY}"
    DEPENDS "${NETCDF_INCLUDE_DIR}/netcdf.h" 
    )

# Utility for checking the compiler CMake actually used

add_executable(print_compiler_version
    print_compiler_version.c
    )

target_include_directories(print_compiler_version
    PUBLIC
    ${MPI_INCLUDE_PATH}
    )
target_link_libraries(print_compiler_version
    PUBLIC
    ${MPI_LIBRARIES}
    )

