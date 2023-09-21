---
title: Building GROMACS in docker - part 1
tags:
 - gromacs
 - docker
 - rse
date: 2020-06-23
---

GROMACS is a versatile package to perform molecular dynamics, i.e. simulate the Newtonian equations of motion for systems with hundreds to millions of particles. GROMACS strongly benefits from hardware optimisation. One of the challenges of using GROMACS in a container is to support this hardware optimisation.

Unpredictale beginnings
=====

Building a hardware optimised version of gromacs is a 2 stage process: first this required that fftw optimised for the specified harware, and then requires that GROMACS be compiled with the hardware optismisation as well.

So how was the container being built? The base container used was the nvidia-docker container which takes care of mapping the GPU device files if present.  Then fftw was compiled with 4 SIMD types flags enabled (SSE2, AVX, AVX2, AVX512). After this GROMACS is compiled in a loop, once for each SIMD type; finally all of these files are copied into a single container with the gmx binary - which will pick the best hardware optimised version on runtime.  This allows the creation of one container that is hardware optimised for as many systems as possible

The problem with this is that the build time is very long, almost 3hrs. There is a little variation each time it is built that means sometimes it takes longer and fails to build on DockerHub due to time limitis.  Such unpredictability is not great for a build system and there is no room to add additional SIMD types.

What if we chain it together?
=====

DockerHub has the ability for a build to be triggered multiple ways.  We were originally using the trigger on push method - when a push is made to the master branch of the github repository, then a build of that branch is triggered on Docker Hub. Another method is to use the REST API to trigger a build. Using this we could create multiple builds on DockerHub. Coupled with DockerHubs ability to use and define variables, we wondered if we could setup a series of builds which would pass the SIMD type as a variable, and trigger each build one after another.

Doing this required adding a post-hook to each docker build that would call a REST API and trigger the next build.  We disabled 'build on push' on all except the first repository in the chain.  This meant that when we pushed changes to the github repo the first build with SIMD set to SSE2 would be triggered, when this finished a post-hook would trigger the next build and so on. The final build triggered would copy the hardware specific GROMACS binaries from the previously created containers and merge them into one container.

This removed the time limit as each build was classed as a separate job with a 3 hour time limit. However it would occasionaly timeout and the build would fail. The setup also meant that a user could not clone the git repo and build their own hardware specific container.

Restructuring the whole thing.
=====

The next attempt was made by colleagues, this involved a lot of changes including python to build the Docker files.  The major overhaul here was the creation of two new repositories, https://github.com/bioexcel/fftw-docker/ and https://github.com/bioexcel/gromacs-docker-container-maker.  The first repositiory contains a Dockerfile to build an optimised containerised version of FFTW; when a git push is maid to this container it triggers a build on DockerHub. The second (gromacs-docker-container-maker) contains multiple directories with a different Dockerfile for each SIMD type and GROMACS version.  When a git push it made to this repository, then multiple builds are triggered on Dockerhub for each Dockerfile. Each Dockerfile builds the specific optimised version of Gromacs whilst copying in the optimised FFTW container built from the fftw-docker repository.  Finally the gromacs-docker repository contains a single dockerfile that combines all of the optimised system specific containers into a single container.

This was a great change that removed the timelimit, and brought much needed changes to the build system. It still suffered from the previous issues of having a chain of containers to build, all of which need to be ran indvidually. If fftw needs updating, then the code needs to be changed in the gromacs-fftw repository and pushed to trigger a build; next, gromacs-docker-container-maker needs to have a build triggered, finally, once that has been completed the final container needs to be rebuilt.  This is not sustainable, and forgetting a step means that it is not updated properly.

GitHub Actions to the rescue
=====

The inability of Dockerhub to support parallel builds or chained builds limited what we could achieve with this.  After looking into a few ways to achieve this we decided to take the dive and see if improvements could be achieved with github actions.

Github actions allows multiple builds, conditionals, and chained steps.  The downside (and upside) is that github actions allows you to do pretty much anything, but you have to build it yourself (or find 3rd party building blocks).

Using several prebuilt actions allowed us to merge all these steps into one (except gromacs-fftw which has not been merged in yet).  Using GH-actions we had a new build chain, instead of having multiple directories holding Dockerfiles, which takes up unnecessary space and requires the maintainer to build and commit the files before they can be built, this was moved into the gh-actions.

Thus, github actions builds the multiple docker files needed, and then builds the containers and registeres them on dockerhub.  Finally a single dockerfile in the repository combines all of these containers into a single container.  Any failures cause the CI to fail, and the maintainers are notified.  Unless all steps complete, a new container is not built which means the whole build chain is always rebuilt if either the gromacs or fftw version is changed, or the SIMD types.

Limitations
=====

So far we have not found a way to separate the variables from the workflow file.  The hope was to have a parameters file in the main repsoitory that the workflow file could then call.  Instead we have had to settle for all the variables being set at the beginning of the workflow file.
 
tl;dr
=====

We cannot have chained and conditional builds on Dockerhub, so use CI to produce the container images and push them to dockerhub instead. We used github actions to build 4 parallel containers, merge them into one, and then register the container on Dockerhub.

Repositories
=====

- https://github.com/longr/gromacs-docker/
- https://github.com/bioexcel/gromacs-docker-container-maker
- https://github.com/bioexcel/fftw-docker/

A lot of work based off:
https://github.com/ahad3112/gromacs-hpccm-recipes-mult-stages
