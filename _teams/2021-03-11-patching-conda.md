---
title: Patching Conda
tags:
 - conda
 - gromacs
date: 2021-03-11
---

Occasionally, there is a need to patch the source code used in Conda. When it is not possible to fix faulty source code directly, you can patch it, by providing a patch file along side the conda build script. This can then modify and patch the source code just before you build the conda package.

There are a few possible reasons for this that I can think of: You have the ability to modify the conda recipe but not the source code; You want to check that patching the source code actually will fix the problem; or, you want to have the conda package created whilst waiting for a patch to make it to the original source code.

This is a short, and not comprehensive description of how to patch faulty source code.  When attempting to package up the latest release of the GROMACS software we found that the Linux build was fine, but the OSX build failed. Eventually we traced the possible cause to a missing header file.  There were some other warnings, but this looked like the cause.  One of the developers asked if we could patch the code to check this was the actual cause of the problem.

Not knowing if it was possible to patch a conda build, or even how to do it, I started searching and stumbled across blog post on Anaconda: [Patching Source Code to Conda Build Recipes](https://www.anaconda.com/blog/patching-source-code-to-conda-build-recipes).  This posted focussed on patching Python packages, and mine was C++, and required use of `conda-build` which I could not get working on my system.  The fundamentals appeared to be:

1. Download the source code.
2. Create a git patch file.
3. Add this and a patch flag to the Conda build.

So I thought I would give this a try - and surprisingly, it worked first time.

The first step, as I mentioned above, was to download the source code.  Now this is specified in the Conda build file:

```yaml
source:
  url: http://ftp.gromacs.org/pub/gromacs/gromacs-{{ version }}.tar.gz
  md5: {{ md5sum }}

```

Where `version` and `md5sum` are variables defined earlier in the `meta.yml`.  Our first step was to download and unzip this source code:

```bash
wget http://ftp.gromacs.org/pub/gromacs/gromacs-2021.tar.gz
tar zxf gromacs-2021.tar.gz
cd gromacs-2021
```

To be able to create a git patch for this code, it needs to be in a git repository. So we initialised a git repository and then made and initial commit.

```bash
git init
git add .
git commit -m 'Initial commit of original source code'
```

The next thing to do was to make the changes needed to the source code, and then create a patch. The change I needed was very simple, and since it affected only OSX, and not Linux it was not going to be easy to test the change on my system - so quite badly, I made the change and assumed it would work.

Once the changes were made and committed to the repository, and then a patch file is created:

```bash
git add -u # add tracked files that have been modified
git commit -m "fix imports"
git format-patch -1
```

This create a file called `0001-fix-imports.patch`.

The next step is to copy this patch to our recipe directory and tell conda to use it. Firstly copy the file `0001-fix-imports.patch` to your recipe directory (in the same location as the `meta.yml` file, and then add the line `patches: 0001-fix-imports.patch` to `meta.yml` in the `source` section:

Our source section then changes from this:

```bash
source:
  url: http://ftp.gromacs.org/pub/gromacs/gromacs-{{ version }}.tar.gz
  md5: {{ md5sum }}
```

to this:

```yaml
source:
  url: http://ftp.gromacs.org/pub/gromacs/gromacs-{{ version }}.tar.gz
  md5: {{ md5sum }}
  patches: 0001-add-header.patch
```

where `0001-add-header.patch` is the name of the patch file that you created.  We then need to add the patch file to our recipe repository and commit the changes to the `meta.yml`

```yaml
git add 0001-add-header.patch
git add meta.yml
git commit -m 'Patching the source code and declaring the patch in the meta.yml'
```

Then `git push` or test locally as needed.  When the build runs, it will download the sourcecode, apply the patch, and then build the software.

