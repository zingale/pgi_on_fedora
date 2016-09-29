How to install the PGI OpenACC Toolkit on Fedora 24

Due to compiler incompatibilities with GCC 6.1, this is a little
involved.

The official installation instructions are here:
http://www.pgroup.com/doc/pgiinstall.pdf

# Install the graphics drivers for your nvidia card

Get this from RPM Fusion.  You can do:

```
yum install akmod-nvidia "kernel-devel-uname-r == $(uname -r)"
```

Reboot so they take effect


# Install the PGI stuff

Untar the PGI compilers and run their install:

```
./install
```

  * EULA is displayed

    Do you accept these terms? (accept,decline) accept

  * Installation directory? [/opt/pgi]

    accept default

  * Install CUDA Toolkit Components? (y/n) y

  * EULA is displayed

    Do you accept these terms? (accept,decline) accept

  * Install AMD software components? (y/n) n

  * Install OpenACC Unified Memory Evaluation package? (y/n) n

  * Do you wish to update/create links in the 2015 directory? (y/n) y

  * Do you wish to install MPICH? (y/n) n

  * Do you wish to generate license keys? (y/n) n


# Setup your environment

```
export PGI=/opt/pgi;
export PATH=/opt/pgi/linux86-64/16.9/bin:$PATH;
export MANPATH=$MANPATH:/opt/pgi/linux86-64/16.9/man;
export LM_LICENSE_FILE=$LM_LICENSE_FILE:/opt/pgi/license.dat;
```


# License

Generate a license at http://pgroup.com using your account

Put the license into `/opt/pgi/license.dat`


# Start the license manager

```
cd $PGI/linux86-64/15.7/bin/
./lmgrd.rc start
```

Maybe also do:

```
cp lmgrd.rc /etc/init.d/lmgrd
ln -s /etc/init.d/lmgrd /etc/rc.d/rc5.d/S90lmgrd
```


# Install CUDA 7.5.x from nvidia

Get their RPM:

https://developer.nvidia.com/cuda-downloads

https://developer.nvidia.com/compute/cuda/8.0/prod/local_installers/cuda-repo-fedora23-8-0-local-8.0.44-1.x86_64-rpm


```
dnf install cuda-repo*
dnf clean all
dnf install cuda
```

This will setup a yum repo and update automatically with your system


# Test it out

```
pgaccelinfo
```

This should tell you about the CUDA capability


# Install older GCC

We need to use an older GCC to get the C++ frontend to work.
Problem is GCC 6.1 cannot compile 5.2, so we need to build
5.4 as an intermediate.


  * wget http://gcc.parentingamerica.com/releases/gcc-5.4.0/gcc-5.4.0.tar.gz

  * `tar xvzf gcc-5.4.0.tar.gz`

  * `cd gcc-5.4.0`

    `./contrib/download_prerequisites`

  * in top dir, mkdir `objdir/`

  * `cd objdir/`

  *  `../gcc-5.4.0/configure --prefix=/opt/gcc/gcc-5.4/ --enable-languages=c,c++,fortran --disable-multilib`

  * `make -j 8`

  * as root, `mkdir /opt/gcc/gcc-5.4`

  * `make install`


# Create a module for older GCC

Create a directory `/etc/modulefiles/gcc`

Create an entry `5.4` in there with the following:

```
#%Module1.0#####################################################################
##
## modules gcc/5.4
##
## modulefiles/gcc/5.4
##
proc ModulesHelp { } {
        global version modroot

        puts stderr "gcc/5.4 - sets the Environment for GCC 5.4 in my home directory"
}

module-whatis   "Sets the environment for using gcc-5.4.0 compilers (C, C++, Fortran)"

# for Tcl script use only
set     topdir          /opt/gcc/gcc-5.4
set     version         5.4
#set     sys             linux86

setenv          CC              $topdir/bin/gcc
setenv          GCC             $topdir/bin/gcc
setenv          CXX             $topdir/bin/g++
setenv          FC              $topdir/bin/gfortran
setenv          F77             $topdir/bin/gfortran
setenv          F90             $topdir/bin/gfortran
prepend-path    PATH            $topdir/include
prepend-path    PATH            $topdir/bin
prepend-path    MANPATH         $topdir/man
prepend-path    LD_LIBRARY_PATH $topdir/lib
```

This allows for

```
module load gcc/5.4
```

# Install GCC 5.2

Do the same procedure to install GCC 5.2, compiling with GCC 5.4


# Tell PGI to use GCC 5.2

To get pgi to recognize this compiler, I need to do:

```
cd /opt/pgi/linux86-64/16.5/bin

makelocalrc `pwd` -gcc /opt/gcc/gcc-5.2/bin/gcc -gpp /opt/gcc/gcc-5.2/bin/g++ -g77 /opt/gcc/gcc-5.2/bin/gfortran -x -net
```

^ that creates a `localrc.machinename`

```
mv localrc localrc.old
```

Make libraries available

```
cd /opt/gcc/gcc-5.2/lib/gcc/x86_64-unknown-linux-gnu/5.2.0

ln -s ../../../../lib64/libgcc_s.so.1 ./libgcc_s.so
ln -s ../../../../lib64/libgfortran.so.3.0.0 ./libgfortran.so
ln -s ../../../../lib64/libgomp.so.1.0.0 ./libgomp.so
ln -s ../../../../lib64/libstdc++.so.6.0.21 ./libstdc++.so
```


# Install MPI and make a module

The Fedora MPICH is already managed via modules.  Here we create one
for MPICH with PGI

Download mpich 3.x

```
CC=pgcc CXX=pgCC FC=pgf95 F77=pgf77 ./configure --prefix=/opt/mpich-pgi
make
make install
```

The last command is done as root

Create `/etc/modulefiles/mpi/mpich-pgi` with

```
#%Module 1.0
#
#  MPICH module for use with 'environment-modules' package:
#

# Only allow one mpi module to be loaded at a time
conflict mpi

# Define prefix so PATH and MANPATH can be updated.
setenv        MPI_BIN       /opt/mpich-pgi/bin
setenv        MPI_SYSCONFIG /etc
setenv        MPI_FORTRAN_MOD_DIR /opt/mpich-pgi/include
setenv        MPI_INCLUDE   /opt/mpich-pgi/include
setenv        MPI_LIB       /opt/mpich-pgi/lib
setenv        MPI_MAN       /opt/mpich-pgi/share/man/mpich
setenv        MPI_COMPILER  mpich-pgi
setenv        MPI_SUFFIX    _mpich
setenv        MPI_HOME      /opt/mpich-pgi
prepend-path  PATH          /opt/mpich-pgi/bin
prepend-path  LD_LIBRARY_PATH /opt/mpich-pgi/lib
prepend-path  MANPATH       /opt/mpich-pgi/share/man/mpich
prepend-path  PKG_CONFIG_PATH /opt/mpich-pgi/lib/pkgconfig
```

To activate MPI for PGI, do:
```
module swap mpi/mpich-x86_64 mpi/mpich-pgi
```

