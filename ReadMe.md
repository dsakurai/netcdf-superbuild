# netcdf-superbuild
Compiles HDF 5 and NetCDF 4. Latter includes C and Fortran bindings.

A simple way of compiling everything at once:
~~{.sh}
$ git submodule update --init --recursive .
$ mkdir whatever_build_directory
$ cd whatever_build_directory
$ export CC=`which mpicc`
$ export CXX=`which mpic++`
$ export FC=`which mpifort`
$ cmake -DCMAKE_INSTALL_PREFIX=some_install_destination ..
~~

Currently assumes MPI to be available.

