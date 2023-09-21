---
title: Latest GROMACS Bioconda release
tags:
 - conda
 
date: 2021-05-19
---

GROMACS is a very powerful widly used piece of software for performing  molecular dynamics, that is, using equations of motion to model systems with hundreds of millions of particles.  Running an analysis using kind of software quickly and efficiently requires a lot of optimisations to the code for the specific hardware.  Whilst GROMACS can be compiled for by users, not everyone will want to do this, and a compromise needs to be found.

A BioConda (an Anaconda/Conda channel) package was created optimised to 4 specific SIMD types to allow users a reasonably optimised package that could be install easily on their machines. When we took over this it had some limitations, namly: No MPI support, and no OSX support.  Over the last few months we have worked to add not only OSX support, but also MPI support.

Adding both of these led to a few problems. The build time for gromacs is small, around 20-30 minutes. However the GROMACS code has to be compiled for each optimised SIMD type separatly. This resulted in a a build time of 80 - 120 minutes per OS. Adding MPI support would treble this (standard build and 2 MPI implementations). An attempt was made to enable MPI support but the builds then timed out, and eventually it was decided to split the 2 packages into `gromacs` and `gromacs_mpi`.


When we came back to tidy up some of the code, it was noticed that a bug in the build process had made the build time longer than it needed to be (it was taking longer than 30 minutes per build).  Once this was removed it was possible to get the build time back down, but we were pushing the limits of the maximum build time.  After a lot of discussion with some of the developers and bioconda core team we decided we could merge this back to a single package with 3 SIMD types as the 4th was too new to be used on most laptops and desktops, and when it is more widly available the lowest common SIMD type will have changed and we can trade them for each other. Additionally it was decided that one of the OpenMPI implementations wold be fine as the second was more developmental. This got our build time down to 120 - 180 minutes.

This has now been merged into the main BioConda channel and a single GROMACS version with optional MPI support is now (available)[https://github.com/bioconda/bioconda-recipes/pull/28377]

We would eventually like to add support for GPUs into this package, however this suffers from 2 issues: Firstly, BioConda does not yet support compiling and building against CUDA libraries; and secondly this would increase the build time again.  Solutions to both of these maybe possible as CondaForge supports both CUDA and a longer build time - so whilst a 2-3hour build time is not ideal, we do have options.