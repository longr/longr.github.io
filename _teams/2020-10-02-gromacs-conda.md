---
title: Creating hardware optimised Conda Packages
tags:
 - conda
 - gromacs
date: 2020-10-02
---

GROMACS is a versatile package to perform molecular dynamics, i.e. simulate the Newtonian equations of motion for systems with hundreds to millions of particles. GROMACS strongly benefits from hardware optimisation.

Conda is a wonderful way of packaging software to make it easily accessible and to support reproducible research. Conda packages can be obtained from different channels, each of which have slightly different methods for building their packages. We used the [bioconda channel](https://bioconda.github.io/). One downside to using Conda is that the code must be compiled for all systems and then packaged - this means it can run on almost any system, but also means that the application is not optimised for the computer it is ran on - for an application like GROMACS optimisation makes a huge difference in the time taken to run an analysis.

Gromacs can and should be optimised for the memory architecture of a system.  This is done at compilation, however for Conda we must compile centrally and then provide binaries. Doing this results in binaries that are not optimised for the system it is running on.

To get round this, we choose to compile 4 versions of GROMACS for the conda package; each optimised for a specific memory architecture. These are compiled into separate directories named after the architecture type:

```bash
/conda/lib.SSE2/*.so
/conda/lib.AVX_256/*.so
/conda/lib.AVX_512/*.so
/conda/lib.AVX2_256/*.so
```

This may not be the best way to achieve this, but it makes development and support easier, and was [agreed with the Conda team](https://github.com/bioconda/bioconda-recipes/pull/19781)

When a use then installs the conda gromacs package, all four binaries are copied over and the `gmx` binary will select which one is needed at runtime. This then allows users to download a the conda package and have an optimised version of gromacs on their machine.

We then realised later that the way the binary checks the system, and how conda packages are accessed meant that the optimised binary was not always selected using this method.  To correct this an *activate* was needed to check the architecture of the system at package activation instead of `gmx` runtime. This was fixed in a later [pull request](https://github.com/bioconda/bioconda-recipes/pull/20149)

So how did we do it?
=====

The first changes is in the build step.  We use a `for` loop to compile each of the **simd types** one at a time and place them into a `bin` and `lib` directory suffixed with their type. (Note `...`, is where I have clipped out lines not needed to illustrate the example.)  

```bash
for ARCH in SSE2 AVX_256 AVX2_256 AVX_512; do \
  cmake .. \
  ...
  -DCMAKE_INSTALL_BINDIR=bin.${ARCH} \
  -DCMAKE_INSTALL_LIBDIR=lib.${ARCH}
  ...
done;


Gromacs then has a script which will check which binary is appropriate for the machine at run time and link to the relevant binary.

`activate.sh`
-----

`activate.sh` is where the work happens.  An `if`, `elif`, `else`, script checks to see which simd types the CPU has and then returns the location of the binary for that simd type.

```bash
DIR="$( dirname "$SOURCE" )"
FLAGS=`cat /proc/cpuinfo | grep ^flags | head -1`
if echo $FLAGS | grep " avx512f " > /dev/null && test -d ${DIR}/../bin.AVX_512 && echo `${DIR}/../bin.AVX_512/identifyavx512fmaunits` | grep "2" > /dev/null; then
    ARCH="AVX_512"
elif echo $FLAGS | grep " avx2 " > /dev/null && test -d ${DIR}/../bin.AVX2_256; then
    ARCH="AVX2_256"
elif echo $FLAGS | grep " avx " > /dev/null && test -d ${DIR}/../bin.AVX_256; then
    ARCH="AVX_256"
else
    ARCH="SSE2"
fi
```

Extra work is needed here as the full path to the binary needs to be given, so a small snippet of code is used to find out exactly where the binary is, more details of how this works can be found on [stackover flow](http://stackoverflow.com/questions/59895/can-a-bash-script-tell-what-directory-its-stored-in/246128#246128)

```bash
SOURCE="${BASH_SOURCE[0]}"
while [ -h "$SOURCE" ]; do # resolve $SOURCE until the file is no longer a symlink
    DIR="$( cd -P "$( dirname "$SOURCE" )" && pwd )"
    SOURCE="$(readlink "$SOURCE")"
    [[ $SOURCE != /* ]] && SOURCE="$DIR/$SOURCE" # if $SOURCE was a relative symlink, we need to resolve it relative to the path where the symlink file was located
done
```

The final line of the script then runs the correct binary passing all arguments to it:

```bash
. ${DIR}/../bin.${ARCH}/GMXRC $@
```


Instead of building the source code into a `bin` and a `lib` dir, we use a `for` loop to put 


Repositories
=====

- https://github.com/bioconda/bioconda-recipes/tree/master/recipes/gromacs/2020
