PnMPI Tool Infrastructure
========================================

[![Travis](https://img.shields.io/travis/LLNL/PnMPI/master.svg?style=flat-square)](https://travis-ci.org/LLNL/PnMPI) [![Codecov](https://img.shields.io/codecov/c/github/LLNL/PnMPI.svg?style=flat-square)](https://codecov.io/github/LLNL/PnMPI?branch=master)  [![](https://img.shields.io/github/issues-raw/LLNL/PnMPI.svg?style=flat-square)](https://github.com/LLNL/PnMPI/issues)
[![GPL license](http://img.shields.io/badge/license-LGPL-blue.svg?style=flat-square)](http://www.gnu.org/licenses/)

by Martin Schulz, schulzm@llnl.gov, LLNL-CODE-402774

### Contributors
 * Todd Gamblin        tgamblin@llnl.gov
 * Tobias Hilbrich     Tobias.Hilbrich@zih.tu-dresden.de
 * Joachim Protze      protze@itc.rwth-aachen.de
 * Felix Muenchhalfen  muenchhalfen@itc.rwth-aachen.de
 * Alexander Haase     alexander.haase@rwth-aachen.de

PnMPI is a dynamic MPI tool infrastructure that builds on top of
the standardized PMPI interface. It allows the user to

 * run multiple PMPI tools concurrently
 * activate PMPI tools without relinking by just changing a
   configuration file
 * multiplex toolsets during a single run
 * write cooperative PMPI tools

The package contains two main components:

 * The PnMPI core infrastructure
 * Tool modules that can explicitly exploit PnMPI's capabilities

So far, this software has mainly been tested on Linux clusters with
RHEL-based OS distributions as well as IBM's BG/P systems.
Some preliminary experiments have also included SGI Altix systems and
Mac OSX 10.6. Ports to other platforms should be straightforward, but
this is not extensively tested. Please contact the author if you run
into problems porting PnMPI or if you successfully deployed PnMPI
on a new platform.

A) Building PnMPI
===============================================================
PnMPI uses CMake for its build system.

A1) Install CMake
-----------------
CMake can be downloaded from http://www.cmake.org.
You will need at least version 2.6, and for BG/P systems you
will need version 2.8.3.

A2) Configure the project
-------------------------
In the simplest case, you can run this in the top-level directory
of the PnMPI tree:

    cmake -DCMAKE_INSTALL_PREFIX=/path/to/install/destination
    make
    make install

This will configure, build, and install PnMPI to the destination
specified.  PnMPI supports parallel make with the -j parameter.
E.g., for using eight build tasks, use:

    cmake -DCMAKE_INSTALL_PREFIX=/path/to/install/destination
    make -j8
    make install

On more complex machines, such as those with filesystems shared among
multiple platforms, you will want to separate out your build directories
for each platform.  CMake makes this easy.

Create a new build directory named according to the platform you
are using, cd into it, an run cmake there.  For example:

    cd <pnmpi>
    mkdir x86_64
    cd x86_64
    cmake -DCMAKE_INSTALL_PREFIX=/path/to/install/destination ..

Here, `<pnmpi>` is the top-level directory in the PnMPI tree.  Note
that when you run CMake this way, you need to supply the path to the
PnMPI *source* directory as the last parameter.  Here, that's just `..`
as we are building in a subdirectory of the source directory.  Once
you run CMake, simply run make and make install as before:

    make -j8
    make install

The PnMPI build should autdetect your MPI installation and determine
library and header locations.  If you want to build with a
particular MPI that is NOT the one autodetected by the build,
you can supply your particular MPI compiler as a parameter:

    cmake \
      -DCMAKE_INSTALL_PREFIX=/path/to/install/destination \
      -DMPI_C_COMPILER=/path/to/my/mpicc \
      ..

See the documentation in [FindMPI.cmake](cmakemodules/legacy/FindMPI.cmake)
for more details on MPI build configuration options.

If you have problems, you may want to build PnMPI in debug mode.  You
can do this by supplying an additional paramter to CMake, e.g.:

    cmake \
      -DCMAKE_INSTALL_PREFIX=/path/to/install/destination \
      -DCMAKE_BUILD_TYPE=Debug \
      ..

The build directory contains a few sample invocations of
cmake that have been successfully used on LLNL systems.


A3) Configuring with/without Fortran
------------------------------------

By default PnMPI is configured to work with C/C++ and
Fortran codes. However, on systems where Fortran is not
available, the system should autodetect this and not
build the Fortran libraries and demo codes. It can also
be manually turned off by adding

    -DENABLE_FORTRAN=OFF

to the cmake configuration command.

The PnMPI distribution contains demo codes for C and
for Fortran that allow you to test the correct
linkage.


### A3a) Configuring with/without Testing, PnMPI-modules

By default PnMPI is configured to build all built-in
modules and demo/test programs. If the build of these
fails they can be switched off by adding

    -DENABLE_DEMO=OFF

resp.

    -DENABLE_MODULES=OFF

to the cmake configuration command.


A4) Configuring for cross-compiled environments
-----------------------------------------------
When configuring PnMPI in cross-compiled environments (such as Blue
Gene/Q systems), it is necessary to provide a matching tool chain
file. Example files that allow the compilation on certain LC machines
can be found in the cmakemodules directory. For example, to configure
PnMPI for a BG/Q machine using the GNU compliler suite, add the
following to the cmake configuration command:

    -DCMAKE_TOOLCHAIN_FILE=../cmakemodules/Toolchain/BlueGeneQ-gnu.cmake

You may need to modify the toolchain file for your system.

A5) Installed structure
-----------------------
Once you've installed, All your PnMPI files and executables should
be in <CMAKE_INSTALL_PREFIX>, the path specified during configuration.
Roughly, the install tree looks like this:

    bin/
      pnmpi                PnMPI invocation tool
      pnmpi-patch          Library patching utility
    lib/
      libpnmpi[f].[so,a]   PnMPI runtime libraries
      pnmpi-modules/       System-installed tool modules
      cmake/               Build files for external modules
    include/
      pnmpi.h              PnMPI header
      pnmpimod.h           PnMPI module support
      pnmpi-config.h       CMake generated configuration file
    share/
      cmake/               CMake files to support tool module builds

Test programs are not installed, but in the [demo](demo) folder of the
build directory, there should also be test programs built with PnMPI.
See below for details on running these to test your PnMPI installation.


A6) Environment setup
-----------------------
You will need to set one environment variable to run PnMPI:

`PNMPI_LIB_PATH` should be set to the full path to the `lib/pnmpi-modules`
directory in your PnMPI installation directory.  This enables the
runtime to find its modules.

Additionally, if you wish to build external modules, you should
set the `PnMPI_DIR` environment variable to the full path to your
PnMPI installation (that is, your `CMAKE_INSTALL_PREFIX`) so that
external module builds can find PnMPI configuration files.  See
Section (C) for more information on building custom modules.

After setting up, for benchmark purposes `PNMPI_BE_SILENT` should be
set, so PnMPI stops producing output.

If you are using modules with app_startup support, but your application does not
use the highest threading level, you may set the reqested MPI threading level
via `PNMPI_THREADING_LEVEL` to decrease the MPI overhead. The level send by
`MPI_Init_thread` will not be sufficient, because MPI will be initialized in the
constructors, if `app_startup` will be used.


### A6a) Using the PnMPI invocation tool

To run PnMPI in front of any application (that is dynamically linked to MPI),
you may use the `bin/pnmpi` tool. It will setup the environment and preloads
PnMPI for you. For a list of supported arguments, invoke it with the `--help`
flag.

    :~$ pnmpi --help
    Usage: pnmpi [OPTION...] utility [utility options]
    P^nMPI -- Virtualization Layer for the MPI Profiling Interface

      -c, --config=FILE          Configuration file
      -q, -s, --quiet, --silent  Don't produce any output
      -?, --help                 Give this help list
          --usage                Give a short usage message
      -V, --version              Print program version
    ...


    :~$ mpiexec -np 2 pnmpi -c pnmpi.conf a.out
    0:
    0:            ---------------------------
    0:           | P^N-MPI Interface         |
    0:           | Martin Schulz, 2005, LLNL |
    0:            ---------------------------
    0:
    0: Number of modules: 4
    0: Pcontrol Setting:  0
    ...


A7) RPATH settings
-----------------------
By default, the build adds the paths of all dependency libraries
to the rpath of the installed PnMPI library.  This is the preferred
behavior on LLNL systems, where many packages are installed and
`LD_LIBRARY_PATH` usage can become confusing.

If you are installing on a system where you do NOT want dependent
libraries added to your RPATH, e.g. if you expect all of PnMPI's
dependencies to be found in system paths, then you can build without
rpath additions using this option to cmake:

    -DCMAKE_INSTALL_RPATH_USE_LINK_PATH=FALSE

This will add only the PnMPI installation lib directory to the
rpath of the PnMPI library.


B) Modules
==========

PnMPI supports two different kind of tool modules:
  * Transparent modules
  * PnMPI-specific modules

Among the former are modules that have been created independently of
PnMPI and are just based on the PMPI interface. To use a transparent
module in PnMPI the user has to perform two steps:
  1. Build the tool as a shared module (a dlopen-able shared library)
  2. Patch the tool using the 'pnmpi-patch' utility, which is incldued
     with the PnMPI distribution.

Usage:

    pnmpi-patch <original tool (in)> <patched tool (out)>

e.g.:

    pnmpi-patch my-module.so my-pnmpi-module.so

After that, copy the tool in the $PNMPI_LIB_PATH directory so that
PnMPI can pick it up.  Note that all of this is handled automatically
by the CMake build files included with PnMPI.

The second option is the use of PnMPI specific modules: these modules
also rely on the PMPI interface, but explicitly use some of the PnMPI
features (i.e., they won't run outside of PnMPI).  These modules
include the "pnmpimod.h" header file. This file also describes the
interface that modules can use. In short, the interface offers an
ability to register a module and after that use a publish/subscribe
interface to offer/use services in other modules. Note: also PnMPI
specific modules have to be patched using the utility described above.

This package includes a set of modules that can be used both
to create other tools using their services and as templates
for new modules.

The source for all modules is stored in separate directories
inside the "module" directory. There are:

* **sample:**
  a set of example modules that show how to wrap send and
  receive operations. These modules are used in the demo
  codes described below.

* **empty:**
  A transparent module that simply wraps all calls without
  executing any code. This can be used to test overhead
  or as a sample for new modules.

* **status:**
  This PnMPI specific module offers a service to add
  extra data to each status object to store additional
  information.

* **requests:**
  This PnMPI specific module offers a service to track
  asynchronous communication requests. It relies on the
  status module.

* **datatype:**
  This PnMPI specific module tracks all datatype
  creations and provides an iterator to walk any datatype.

* **comm:**
  This PnMPI specific module abstracts all communication in
  a few simple callback routines. It can be used to write
  quick prototype tools that intercept all communication
  operations independent of the originating MPI call.
  This infrastructure can be used by creating submodules: two
  such submodules are included: an empty one that can be used
  as a template and one that prints every message.
  Note: this module relies on the status, requests, and
  datatype modules. A more detailed description on how to
  implemented submodules is included in the comm directory
  as a separate README.

* **metrics-counter**
  This PnMPI specific module counts the MPI call invocations. Add the module
  at the top of your config file to count how often each rank invoked which MPI
  call or in front of a specific module to count how often invocations reached
  this module.

* **metrics-timing**
  This PnMPI specific module measures the time MPI call invocations take. It has
  two different operation modes:

  * simple: Add it in front of a module and it will measure the time of all
    following modules.

    ```
    module metrics-timing
    module sample1
    ```

  * advanced: Add the timing module before and after the modules you want to
    measure and it will only measure the time of these, but not the following
    modules.

    ```
    module metrics-timing
    module sample1
    module sample2
    module metrics-timing

    module empty
    ```

    Measuring `MPI_Pcontrol` is available in advanced mode, only. To measure
    `MPI_Pcontrol` calls both `metrics-timing` invocations need to be pcontrol
    enabled:

    ```
    module metrics-timing
    pcontrol on # May be ignored if metrics-timing is the first module.
    module sample1
    module metrics-timing
    pcontrol on
    ```

* **wait-for-debugger**
  This module prints the PID of each rank before executing the application. If
  the `WAIT_AT_STARTUP` environment variable is set to a numeric value, the
  execution will be delayed up to `value` seconds, so you may attach with a
  debugger in that time.

Note: All modules should be compiled with mpicc or equivalent
(which includes the MPI header files) and should be linked
without linking to the MPI library (to avoid MPI routines
being linked to multiple times).


C) Building your own modules with CMake
=======================================
PnMPI installs CMake build files with its distribution to allow
external projects to quickly build MPI tool modules.  The build
files allow external tools to use PnMPI, the pnmpi-patch utility,
PnMPI's wrapper generator, and PnMPI's dependency libraries.

To create a new PnMPI module, simply create a new project that looks
something like this:

    my-project/
      CMakeLists.txt
      foo.c
      wrapper.w

Assume that wrapper.w is a wrapper generator input file that will
generate another file called wrapper.c, which contains MPI interceptor
functions for the tool library.  foo.c is additional code needed for
the tool library.  CMakeLists.txt is the CMake build file.  Before
you build, you will need to set the PnMPI_DIR environment variable:

    setenv PnMPI_DIR /path/to/pnmpi/installation

Unless you have installed PnMPI in a standard system location (/usr,
/usr/local, etc.), in which case it will be discoveredautomatically.

Your CMakeLists.txt file should start start with something like this:

```CMake
project(my-module C)
cmake_minimum_required(VERSION 2.8)

find_package(PnMPI REQUIRED)
find_package(MPI REQUIRED)

add_wrapped_file(wrapper.c wrapper.w)
add_pnmpi_module(foo foo.c wrapper.c)

install(TARGETS foo DESTINATION ${PnMPI_MODULES_DIR})

include_directories(
  ${MPI_INCLUDE_PATH}
  ${CMAKE_CURRENT_SOURCE_DIR})
```

`project()` and `cmake_minimum_required()` are standard.  These tell the build
the name of the project and what version of CMake it will need.

`find_package(PnMPI)` will try to find PnMPI in system locations, and if
it is not found, it will fail because the `REQUIRED` option has been specified.

`find_package(MPI)` locates the system MPI installation so that you can use its
variables in the `CMakeLists.txt` file.

`add_wrapped_file()` tells the build that `wrapper.c` is generated from
`wrapper.w`, and it adds rules and dependencies to the build to automatically
run the wrapper generator when `wrapper.c` is needed.

Finally, `add_pnmpi_module()` function's much like CMake's builtin
`add_library()` command, but it ensures that the PnMPI library is built as a
loadable module and that the library is properly patched.

`install()` functions just as it does in a standard CMake build, but here we
install to the `PnMPI_MODULES_DIR` instead of an install-relative location.
This variable is set if PnMPI is found by `find_package(PnMPI)`, and it allows
users to install directly into the system PnMPI installation.

Finally, `include_directories()` functions as it would in a normal CMake build.

Once you've make your `CMakeLists.txt` file like this, you can build your
PnMPI module like so:

    cd my-module
    mkdir $SYS_TYPE && cd $SYS_TYPE
    cmake ..
    make -j8
    make -j8 install

This should find PnMPI on your system and build your module, assuming that
you have your environment set up correctly.

### C2) Limiting the threading level

If your module is not threadsafe or is only able to process a limited amount of
threading, it may provide an integer named `PnMPI_threading_level` to define the
maximum provided threading level of this module.

### C3) Module hooks

At different points hooks will be called in all loaded modules. These can be
used to trigger some functionality at a given time. All hooks have the return
type `void`.

* `PNMPI_RegistrationPoint`: This hook will be called just after the module has
  been loaded. It may be used to register the name of the module, services
  provided by the module, etc.
* `PNMPI_RegistrationComplete`: This hook will be called after PnMPI is
  initialized and all modules have been registered.
* `app_startup`: If a module provides an `app_startup` hook, PnMPI will
  initialize MPI in the applications constructor *before* `main` is started and
  call the hook.
* `app_shutdown`: If a module provides an `app_shutdown` hook, PnMPI will not
  call `PMPI_Finalize` in the `MPI_Finalize` wrapper, but keeps MPI open until
  `main` has finished and calls the hook in the applications destructor. After
  the hooks of all modules have been called, MPI will be shut down.


D) Debug Options
================
The default build of PnMPI includes debug print options, that can be dynamically
enabled. To control it, the environment variable 'PNMPI_DBGLEVEL' can be set to
any combination of the following debug levels:

* `0x01` - Messages about PnMPI initialization (before MPI is initialized).
* `0x02` - Messages about module loading.
* `0x04` - Messages about MPI call entry and exit.

**NOTE:** The first two levels should be enabled for single-rank executions
only, as their output can't be limited to a single rank and thus will be printed
on **all** ranks.

Additionally, the printouts can be restricted to a single node by setting the
variable `PNMPI_DBGNODE` to an MPI rank.

### Using the PnMPI invocation tool

You may set the above options in the PnMPI invocation tool. Use the `--debug`
option to enable a specific debug level and `--debug-node` to limit the debug
output to a single rank.


E) Configuration and Demo codes
===============================

The PnMPI distribution includes two demo codes (in C and F77).
They can be used to experiment with the basic PnMPI functionalities
and to test the system setup. The following describes the
C version (the F77 version works similarly):

1. change into the [demo](demo) directory
2. The program [simple.c](demo/simple.c), which sends a message from any task
  with ID>0 to task 0, was compiled into three binaries:
    * simple    (plain MPI code)
    * simple-pn (linked with PnMPI)
    * simple-s1 (plain code linked with sample1.so)

3. executing simple will run as usual
  The program output (for 2 nodes) will be:

       GOT 1

4. by relinking the code, one can use any of the
   original (unpatched) modules with this codes.
   The unpatched modules are in modules/sample
   Examples: simplest-s1 linked with sample1

5. PnMPI is configured through a configuration file that lists all modules
   to be load by PnMPI as well as optional arguments. The name for this
   file can be specified by the environment variable `PNMPI_CONF`. If this
   variable is not set or the file specified can not be found, PnMPI looks
   for a file called `.pnmpi_conf` in the current working directory, and
   if not found, in the user's home directory.

   By default the file in the demo directory is named `.pnmpi_conf` and
   looks as follows:

       module sample1
       module sample2
       module sample3
       module sample4

  (plus some additional lines starting with #, which indicates comments)

  This configuration causes these four modules to be loaded
  in the specified order. PnMPI will look for the corresponding
  modules (.so shared library files) in `PNMPI_LIB_PATH`.

6. Running simple-pn will load all four modules in the specified
   order and intercept all MPI calls included in these modules:
    * sample1: send and receive
    * sample2: send
    * sample3: receive
    * sample4: send and receive
  The program output (for 2 nodes) will be:

          0:
          0:            ---------------------------
          0:           | P^N-MPI Interface         |
          0:           | Martin Schulz, 2005, LLNL |
          0:            ---------------------------
          0:
          0: Number of modules: 4
          0: Pcontrol Setting:  0
          0:
          0: Module sample1: not registered (Pctrl 1)
          0: Module sample2: not registered (Pctrl 0)
          0: Module sample3: not registered (Pctrl 0)
          0: Module sample4: not registered (Pctrl 0)
          0:
          WRAPPER 1: Before recv
          WRAPPER 1: Before send
          WRAPPER 3: Before recv
          WRAPPER 4: Before recv
          WRAPPER 2: Before send
          WRAPPER 4: Before send
          WRAPPER 4: After recv
          WRAPPER 3: After recv
          WRAPPER 1: After recv
          WRAPPER 4: After send
          WRAPPER 2: After send
          WRAPPER 1: After send
          WRAPPER 1: Before recv
          WRAPPER 3: Before recv
          WRAPPER 1: Before send
          WRAPPER 2: Before send
          WRAPPER 4: Before send
          WRAPPER 4: After send
          WRAPPER 2: After send
          WRAPPER 1: After send
          GOT 1
          WRAPPER 4: Before recv
          WRAPPER 4: After recv
          WRAPPER 3: After recv
          WRAPPER 1: After recv

  When running on a BG/P systems, it is necessary to explicitly export
  some environment variables. Here is an example:

        mpirun -np 4 -exp_env LD_LIBRARY_PATH -exp_env PNMPI_LIB_PATH -cwd $PWD simple-pn


F) Using `MPI_Pcontrol`
========================
The MPI standard defines the `MPI_Pcontrol`, which does not have any
direct effect (it is implemented as a dummy call inside of MPI), but
that can be replaced by PMPI to accepts additional information from
MPI applications (e.g., turn on/off data collection or markers for
main iterations). The information is used by a PMPI tool linked to
the application. When it is used with PnMPI the user must therefore
decide which tool an `MPI_Pcontrol` call is directed to.

By default PnMPI will direct Pcontrol calls to first module in the
tool stack only. If this is not the desired effect, users can turn
on and off which module Pcontrols reach by adding `pcontrol on` and
`pcontrol off` into the configuration file in a separate line
following the corresponding module specification. Note that PnMPI
allows that Pcontrol calls are sent to multiple modules.

In addition, the general behavior of Pcontrols can be specified
with a global option at the beginning of the configuration file.
This option is called "globalpcontrol" and can take one of the
following arguments:

* `int` 	only deliver the first argument, but ignore
	the variable list arguments (default)
* `pmpi`	forward the variable list arguments
* `pnmpi`	requires the application to specify a specific
  format in the variable argument list known to
 	PnMPI (level must be PNMPI_PCONTROL_LEVEL)
* `mixed` same as pnmpi, if level==PNMPI_PCONTROL_LEVEL
   same as pmpi,  if level!=PNMPI_PCONTROL_LEVEL
* `typed  	<tlevel> <type>`
	forward the first argument in any case, the
	second only if the level matches <tlevel>. In
	this case, assume that the second argument is
	of type <type>

The PnMPI internal format for Pcontrol arguments is as follows:

    int level (same semantics as for MPI_Pcontrol itself)
    int type = PNMPI_PCONTROL_SINGLE or PNMPI_PCONTROL_MULTIPLE
                   (target one or more modules) |
                   PNMPI_PCONTROL_VARG or PNMPI_PCONTROL_PTR
                   (arguments as vargs or one pointer)
    int mod = target module (if SINGLE)
    int modnum = number of modules (if MULTIPLE)
    int *mods = pointer to array of modules
    int size = length of all variable arguments (if VARG)
    void *buf = pointer to argument block (if PTR)

Known issues:

Forwarding the variable argument list as done in pmpi and mixed is only
implemented in a highly experimental version and disabled by default.
To enable, compile PnMPI with the flag `EXPERIMENTAL_UNWIND` and link
PnMPI with the libunwind library. Note that this is not extensively
tested and not portable across platforms.


G) References
=============
More documentation on PnMPI can be found in the following two
published articles:

Martin Schulz and Bronis R. de Supinski.
**PnMPI Tools: A Whole Lot Greater Than the Sum of Their Parts**.
*Supercomputing 2007*.
November 2007, Reno, NV, USA.

Available at: http://sc07.supercomputing.org/schedule/pdf/pap224.pdf

Martin Schulz and Bronis R. de Supinski.
**A Flexible and Dynamic Infrastructure for MPI Tool Interoperability**.
*International Conference on Parallel Processing (ICPP)*.
August 2005, Columbus, OH, USA.
Published by IEEE Press

Available at: http://ieeexplore.ieee.org/iel5/11126/35641/01690620.pdf?isnumber=35641&prod=CNF&arnumber=1690620&arSt=193&ared=202&arAuthor=Martin+Schulz%3B+Bronis+R.+de+Supinski


H) Contact
==========
For more information or in case of questions, please contact
Martin Schulz at schulzm@llnl.gov.


Copyright
===========
Copyright (c) 2008-2014, Lawrence Livermore National Security, LLC.
All rights reserved - please read information in "LICENCSE"
