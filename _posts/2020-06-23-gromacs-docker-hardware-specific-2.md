---
title: Building GROMACS in docker part 2
tags:
 - gromacs
 - docker
 - rse
date: 2020-06-23
---

In the previous post we discussed how the gromacs docker container was originally made, and what option we went through to improve the workflow finally settling on GitHub Actions.  In this part we will describe how we did this.

The build consists of 4 steps:

1. Get the build parameters (GROMACS version, SIMD types and so on)
2. Build a container containing FFTW optimised for all SIMD types.
3. Build a GROMACS container optimised for each SIMD type, using the FFTW container.
4. Combine all optimised GROMACS container into one container.


Get the build parameters
=====

This took the most amount of work - using `gromacs-hpccm-recipes-mult-stages` we are able to build Dockerfiles for almost any configuration of GROMACS we can think of.  Parametrising most of these variables was simple, they can be specified at the beginning of the file and used later.

Variables can be defined and set in a step, job. Additionally they can be defined at the beginning of a workflow file, making them global to the file.  Here we defined the variable `gromacs` to hold the gromacs version.

```yaml
---
name: Build and push to Docker Hub
on: [push,pull_request]
env:
  gromacs: 2020.2
```

The variables can then be retrieved by calling them as so:

```yaml
${{env.gromacs}}
```

Using these variables, we can specify all the variables we need for creating Dockerfiles.

```yaml
---
name: Build and push to Docker Hub
on: [push,pull_request]
env:
  # docker_repo:   This must be changed between forks.  This should be the
  #                dockerhub repository you will be using to register the
  #                docker containers to.
  docker_repo: longr/gromacs-docker

  # Here you can specify the versions of the
  # various parameters, such as gcc or cuda.
  fftw: 3.3.8
  fftw_md5: 8aac833c943d8e90d51b697b27d4384d
  gromacs: 2020.2
  cmake: 3.17.1
  gcc: 8
  cuda: 10.1
  openmpi: 4.0.0
  ubuntu: 18.04
  rdtscp: off

  # additional_simd_types:   This is where you need to specify any
  #                          SIMD types that you want to build a
  #                          container for.
  additional_simd_types: "sse2 avx avx2"
```

Next we need to create a Docker container for the optimised version of FFTW.  We can create a file for this quite simply:

```bash
FROM ubuntu:18.04

ARG FFTW_VERSION=3.3.8
ARG FFTW_MD5=8aac833c943d8e90d51b697b27d4384d

# install required packages
RUN apt-get update \
  && apt-get install -y --no-install-recommends \
    software-properties-common \
  && add-apt-repository ppa:ubuntu-toolchain-r/test \
  && apt-get update \
  && apt-get install -y --no-install-recommends \
    build-essential \
    curl \
    gcc-9 \
  && rm -rf /var/lib/apt/lists/*

# Install fftw with more optimizations than the default packages.  It
# is not critical to run the tests here, since the GROMACS tests will
# catch fftw build errors too.

RUN curl -o fftw.tar.gz http://www.fftw.org/fftw-${FFTW_VERSION}.tar.gz \
  && echo "${FFTW_MD5}  fftw.tar.gz" > fftw.tar.gz.md5 \
  && md5sum -c fftw.tar.gz.md5 \
  && tar -xzvf fftw.tar.gz && cd fftw-${FFTW_VERSION} \
  && ./configure --disable-double --enable-float --disable-static --enable-shared \
  && make \
  && make install \
  && rm -rf fftw.tar.gz fftw.tar.gz.md5 fftw-${FFTW_VERSION}
```

The problem with this Dockerfile is that the FFTW version, and SIMD types are hard coded.  To remove these we can use `sed` in the workflow file:

```yaml
  # Builds the fftw container needed for the final combined container.
  build_fftw_container:
    runs-on: ubuntu-18.04
    if: "!contains(github.event.head_commit.message, 'ci skip')"
    steps:
    - uses: actions/checkout@master

    # Find and replace default fftw with version specified at top

    - name: Set FFTW version
      run: |
        sed -i "s/3.3.8/${{env.fftw}}/g" fftw/Dockerfile
        sed -i "s/8aac833c943d8e90d51b697b27d4384d/${{env.fftw_md5}}/g" fftw/Dockerfile
	sed -i "s/18.04/${{ env.ubuntu }}
```

Here we use `sed` to replace the hard coded values with the variables we specified at the beginning of the file.  The next thing we need to do is add the optimisation flags to the compilation line. We do this with sed again.  We need an extra part to the command this time.  The normal `sed` syntax for find and replace is

```bash
sed -i "s/<find>/<replace>/g" <file>
```

Instead of doing this, by adding `&` at the beginning of the replacement string we tell `sed` that instead of replacing the found expression, it should be appended to the end.

```bash
sed -i "s/<find>/&<replace>/g" <file>
```

This way we can search for `./configure` and add an optimisation flag `--enable-simd_type` which results in `./configure --enable-simd_type`.  If we do this once per simd type we can build fftw to be optimised for all the simd types. We can do this using a `for` loop.


```bash
    # Loop over simd types and append to build command in Dockerfile.
    - name: Add SIMD types
      run: |
        for type in ${{env.additional_simd_types}}
        do
            sed -i "s/configure/& --enable-$type /g" fftw/Dockerfile
        done
```

The last thing we then need to do to create the FFTW container is to build and push the docker container to DockerHub.  We split this into 3 steps: First, build the container from our now modified Dockerfile; Secondly, Login to DockerHub using the CLI; and finally push the container to DockerHub.

```yaml
    - name: Build fftw container
      run: |
        docker build -t "${{env.docker_repo}}:fftw-${{env.fftw}}" -f fftw/Dockerfile .
    - name: Docker Login
      run: docker login -u ${{secrets.DOCKER_USERNAME}} -p ${{secrets.DOCKER_PASSWORD}}

    - name: Docker Push
      if: "${{ github.event_name == 'push' }}"
      run: |
        docker push "${{env.docker_repo}}:fftw-${{env.fftw}}"
        sleep 60
        # Needed to give time to register container before being
        # Pulled by next steps.
```

Building optimised containers
=====

The next thing we need to do is create and build the optimised containers for each SIMD type.  GitHub Actions allows us to create a matrix of build options to be used in a job.  We can create a `key: value` pair and assign it an array of values.

```yaml
strategy:
  matrix:
   simd_type: ['avx', 'sse2']

steps:
  name: Build Container
  run: Build ${{ matrix.simd_type }}
```

This is how we first created this, but this required modifying the workflow file in multiple locations which was undesirable.  A solution to this was found with [fromJSON](https://github.blog/changelog/2020-04-15-github-actions-new-workflow-features/) which allows a JSON to be used instead of `key: value` pairs. To specify the JSON you need to use the `echo` command in a job. The example github use is:

```yaml
name: build
on: push
jobs:
  job1:
    runs-on: ubuntu-latest
    outputs:
      matrix: ${{ steps.set-matrix.outputs.matrix }}
    steps:
    - id: set-matrix
      run: echo "::set-output name=matrix::{\"include\":[{\"project\":\"foo\",\"config\":\"Debug\"},{\"project\":\"bar\",\"config\":\"Release\"}]}"
  job2:
    needs: job1
    runs-on: ubuntu-latest
    strategy:
      matrix: ${{fromJson(needs.job1.outputs.matrix)}}
    steps:
    - run: build
```
This is a complex statement to edit, but as it is just a string we can create it from a much simpler variable, the one we specified earlier `additional_simd_types`.

We now need to create a job that will create and export the JSON we need from our input variable:

```yaml
  get_builds:
    runs-on: ubuntu-18.04
    outputs:
      matrix: ${{ steps.set-matrix.outputs.matrix }}
    steps:
    - id: set-matrix
      run: |
        for type in ${{ env.additional_simd_types }}; do SIMD=$SIMD{\"simd\":\"$type\"}; done;
        SIMD=`echo $SIMD | sed 's/}{/},{/g'`
        echo "::set-output name=matrix::{\"include\":[$SIMD]}"
```

This job creates the JSON which can then be used as an input to the build job:

```yaml
# Build sub containers, one for each SIMD type
  build_subcontainer:
    needs: [build_fftw_container, get_builds]
    # Fetch JSON created from additional_simd_types
    strategy:
      matrix: ${{fromJson(needs.get_builds.outputs.matrix)}}
    runs-on: ubuntu-18.04
```

To create the Dockerfiles for our optimised containers we need two things: Python and `gromacs-hpccm-recipes-mult-stages` which is kept as a submodule in our repository, as such we need to pass an option to `checkout` to checkout the submodule as well.

```yaml
    if: "!contains(github.event.head_commit.message, 'ci skip')"
    steps:
    - uses: actions/checkout@v2
      with:
        submodules: 'true'
```

We then need to checkout Python

```yaml
    - uses: actions/setup-python@v1
      with:
        python-version: "3.7"

    - name: Install python dependencies
      run: |
        set -xe
        python -VV
        python -m site
        python -m pip install --upgrade pip
        python -m pip install hpccm
```

We now need to create the Dockerfile using `gromacs-hpccm-recipes-mult-stages`:

```yaml
   # The Dockerfiles must be generated based on SIMD type, Gromacs
   # version and CUDA version
    - name: Generate Dockerfiles
      env:
        docker_tag: gmx-${{env.gromacs}}-cuda-${{env.cuda}}-${{matrix.simd}}
      run: |
        cd gromacs-hpccm-recipes-mult-stages
        python3 generate_specifications_file.py \
        --format docker \
        --gromacs ${{env.gromacs}} \
        --ubuntu ${{env.ubuntu}} \
        --gcc ${{env.gcc}} \
        --cuda ${{env.cuda}} \
        --cmake ${{env.cmake}} \
        --engines simd=${{matrix.simd}}:rdtscp=${{env.rdtscp}} \
        --fftw-container ${{env.docker_repo}}:fftw-${{env.fftw}} \
        --regtest \
        > Dockerfile
```

Finally we need to build and register the Docker container as we did for FFTW:

```yaml

    - name: Build the Docker image
      env:
        docker_tag: gmx-${{env.gromacs}}-cuda-${{env.cuda}}-${{matrix.simd}}
      run: |
        cd gromacs-hpccm-recipes-mult-stages
        docker build -t "${{env.docker_repo}}:${{env.docker_tag}}" -f Dockerfile .
    - name: Docker Login
      run: docker login -u ${{secrets.DOCKER_USERNAME}} -p ${{secrets.DOCKER_PASSWORD}}

    - name: Docker Push
      env:
        docker_tag: gmx-${{env.gromacs}}-cuda-${{env.cuda}}-${{matrix.simd}}
      if: "${{ github.event_name == 'push' }}"
      run: |
        docker push "${{env.docker_repo}}:${{env.docker_tag}}"
```

This then builds and register one optimised gromacs container for simd type specified `additional_simd_types`.


Creating a combined container
=====

As with the FFTW container, we have a pre-built Dockerfile:

```bash
FROM nvidia/cuda:10.2-runtime-ubuntu18.04

# install required packages
RUN apt-get update \
  && apt-get install -y --no-install-recommends \
    libgomp1 \
    liblapack3 \
    openmpi-bin \
    openmpi-common \
    python3 \
  && rm -rf /var/lib/apt/lists/*

## Add the fftw3 libraries
#COPY --from=gromacs/fftw /usr/local/lib /usr/local

# Add the GROMACS configurations

# Add architecture-detection script
COPY gmx-chooser /gromacs/bin/gmx
RUN chmod +x /gromacs/bin/gmx

ENV PATH=$PATH:/gromacs/bin
```

We first need to change the Ubuntu versions and CUDA version to those specified in a variables at the beginning:

```yaml
# Combine all containers into one primary container and
# publish to docker hub
  build_final_container:
    needs: build_subcontainer
    runs-on: ubuntu-18.04
    if: "!contains(github.event.head_commit.message, 'ci skip')"

    # Only combine and push to Docker Hub if we are on dev branch (TODO: master) and 
    # this is not a pull request. Skip if commit message is "ci skip"
    steps:
    - uses: actions/checkout@master

    - name: Edit Dockerfile- Set repo
      run: |
        sed -i "s|gromacs/gromacs-docker|${{env.docker_repo}}|g" Dockerfile
    - name: Edit Dockerfile- Set CUDA version
      run: |
        sed -i "s|FROM nvidia/cuda:10.2|FROM nvidia/cuda:${{env.cuda}}|g" Dockerfile
    - name: Edit Dockerfile- Set Ubuntu version
      run: |
        sed -i "s|runtime-ubuntu18.04|runtime-ubuntu${{env.ubuntu}}|g" Dockerfile
```

We then need to add the lines to tell Docker to copy the gromacs binaries from our optimised containers. We do this by looping over `additional_simd_types` and using the `a` command in sed.

```bash
sed -i 's/<find>/a <replace>/g'
```
This tells `sed` to find any instance of 'find' and insert 'replace' on the line after.  You will also notice that instead of using '/' as the delimiter in `sed` we have used `|`.  `sed` does not care what is used as the delimiter, and by using `|` we don't have to escape every forward slash that we use.

```yaml
    - name: Edit Dockerfile- Set SIMD types to be loaded
      run: |
        for type in ${{env.additional_simd_types}}
        do
            sed -i "s|GROMACS configurations|a COPY --from=gromacs/gromacs-docker:gmx-${{env.gromacs}}-cuda-${{env.cuda}}-$type     /gromacs /gromacs|g" Dockerfile
        done


Finally we need to build and publish the docker container:

```yaml
    - name: Build the combined Docker image
      env:
        docker_tag: gmx-${{env.gromacs}}-cuda-${{env.cuda}}
      run: |
        docker build -t "${{env.docker_repo}}:${{env.docker_tag}}" -t "${{env.docker_repo}}:latest" -f Dockerfile .
    - name: Docker Login
      run: docker login -u ${{secrets.DOCKER_USERNAME}} -p ${{secrets.DOCKER_PASSWORD}}

    - name: Docker Push version tag
      env:
        docker_tag: gmx-${{env.gromacs}}-cuda-${{env.cuda}}
      run: |
        docker push "${{env.docker_repo}}:${{env.docker_tag}}"
```

This should then build and tag the Docker container, and push it to docker hub.  Experience (and the lines in the code to ensure it) suggests there is a delay between pushing and being able to pull the container of about a minute.