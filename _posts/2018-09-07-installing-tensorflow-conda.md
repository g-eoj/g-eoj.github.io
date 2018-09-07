---
layout: post
title: Installing a TensorFlow Wheel in a Conda Environment
tags: [conda, tensorflow]
---

Now that we have a wheel after following the official TensorFlow build instructions, how do we safely install it in a Conda environment?
<!--more--> 
Let's assume we have the following wheel: `~/tf-builds/wheels/tensorflow-1.10.1-cp36-cp36m-linux_x86_64.whl`.
We want to install it in a Conda environment.

There are two ways to do this:
1. Install the wheel with pip
2. Create a local Conda package from the wheel

### Gather Dependencies

Both methods require installing the wheel itself with a `--no-deps` flag to avoid clobbering the Conda environment[^1].
So the first thing to do is to find the wheel's dependencies.
This can be done with `pkginfo`, which is available through Conda.

    $ pkginfo ~/tf-builds/wheels/tensorflow-1.10.1-cp36-cp36m-linux_x86_64.whl
    metadata_version: 2.1
    name: tensorflow
    version: 1.10.1
    platforms: ['UNKNOWN']
    summary: TensorFlow is an open source machine learning framework for everyone.
    description: TensorFlow is an open source software library for high performance numerical
    computation. Its flexible architecture allows easy deployment of computation
    across a variety of platforms (CPUs, GPUs, TPUs), and from desktops to clusters
    of servers to mobile and edge devices.

    Originally developed by researchers and engineers from the Google Brain team
    within Google's AI organization, it comes with strong support for machine
    learning and deep learning and the flexible numerical computation core is used
    across many other scientific domains.



    keywords: tensorflow tensor machine learning
    home_page: https://www.tensorflow.org/
    author: Google Inc.
    author_email: opensource@google.com
    license: Apache 2.0
    classifiers: ['Development Status :: 5 - Production/Stable', 'Intended Audience :: Developers', 'Intended Audience :: Education', 'Intended Audience :: Science/Research', 'License :: OSI Approved :: Apache Software License', 'Programming Language :: Python :: 2', 'Programming Language :: Python :: 2.7', 'Programming Language :: Python :: 3', 'Programming Language :: Python :: 3.4', 'Programming Language :: Python :: 3.5', 'Programming Language :: Python :: 3.6', 'Topic :: Scientific/Engineering', 'Topic :: Scientific/Engineering :: Mathematics', 'Topic :: Scientific/Engineering :: Artificial Intelligence', 'Topic :: Software Development', 'Topic :: Software Development :: Libraries', 'Topic :: Software Development :: Libraries :: Python Modules']
    download_url: https://github.com/tensorflow/tensorflow/tags
    requires_dist: ['absl-py (>=0.1.6)', 'astor (>=0.6.0)', 'gast (>=0.2.0)', 'numpy (<=1.14.5,>=1.13.3)', 'six (>=1.10.0)', 'protobuf (>=3.6.0)', 'setuptools (<=39.1.0)', 'tensorboard (<1.11.0,>=1.10.0)', 'termcolor (>=1.1.0)', 'grpcio (>=1.8.6)', 'wheel (>=0.26)']

The `requires_dist` section lists the dependencies, along with version numbers.
The required Python version is not listed, however you can infer this from the file name of the wheel.
For this example the required Python version is 3.6.
Another source for checking dependencies is the [conda-forge feedstock recipe](https://github.com/conda-forge/tensorflow-feedstock/blob/master/recipe/meta.yaml), though it may not apply if the wheel was built from the master branch.

Now that we have the dependencies we can move on to one of the two installation methods.


### Install the Wheel with pip

Use `conda` to manually install all the dependencies in the Conda environment. Then, with the environment active, run:

    pip install --no-deps ~/tf-builds/wheels/tensorflow-1.10.1-cp36-cp36m-linux_x86_64.whl


### Create a Local Conda Package

Install `conda-build` in the base Conda environment. 
As far as I can tell, `conda-build` has to be run from the base environment to make local packages accessible  with the `--use-local` argument.
It is possible to install with an absolute path to the package, however I had issues with requirements not being installed if I did things that way.

Next, make a `meta.yaml` file. 
The [conda-forge feedstock recipe](https://github.com/conda-forge/tensorflow-feedstock/blob/master/recipe/meta.yaml) is a good starting point for how to setup the `meta.yaml` file, although again, the conda-forge recipe may not be correct for the specific branch the wheel was built from.
In the example below, `script:` is used in the `build:` section so that there's no need for a separate `build.sh` file.
In this case, "vanilla" is used as a build string to designate that the wheel was built without any special build options.

    package:
      name: tensorflow
      version: "1.10.1"

    build:
      script: python -m pip install --no-deps ~/tf-builds/wheels/tensorflow-1.10.1-cp36-cp36m-linux_x86_64.whl
      string: "vanilla"
      entry_points:
        - freeze_graph = tensorflow.python.tools.freeze_graph:run_main
        - toco_from_protos = tensorflow.contrib.lite.toco.python.toco_from_protos:main
        - tflite_convert = tensorflow.contrib.lite.python.tflite_convert:main
        - toco = tensorflow.contrib.lite.python.tflite_convert:main
        - saved_model_cli = tensorflow.python.tools.saved_model_cli:main

    requirements:
      build:
        - pip
        - python
      run:
        - python 3.6.*
        - absl-py>=0.1.6
        - astor>=0.6.0
        - gast>=0.2.0
        - numpy>=1.13.3
        - six>=1.10.0
        - protobuf>=3.6.0
        - tensorboard<1.11.0,>=1.10.0
        - termcolor>=1.1.0
        - grpcio>=1.8.6

Switch to the base Conda environment. 
In the same directory as `meta.yaml` run:

    conda-build .


#### Installing the Package
The above package can be installed in any Conda environment with:

    conda install --use-local tensorflow=1.10.1=*vanilla*

Notice how the build string can be specified. 
It's possible to make individual packages for the same version of TensorFlow with different build options. 
Then a build string can be used to choose which build options are desired when the package is installed.
Since the package name and version remains the same, the package will still satisfy requirements of other packages.

This is one of the big advantages of making a Conda package. The other advantage being it will install dependencies for you.

[^1]: [Using wheel files with conda](https://conda.io/docs/user-guide/tasks/build-packages/wheel-files.html)
