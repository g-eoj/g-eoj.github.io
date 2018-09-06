---
layout: post
title: Building TensorFlow 1.10.1 from Source
tags: [tensorflow, keras, conda]
---

This post documents what I did to build TensorFlow 1.10.1 from source. 
Based on the official [install guide](https://www.tensorflow.org/install/install_sources).
<!--more-->

Setup Build Environment:

    conda create -n tf-build python=3.6 bazel numpy
    conda activate tf-build

Clone TensorFlow repository and checkout current version:

    git clone https://github.com/tensorflow/tensorflow
    cd tensorflow
    git checkout v1.10.1

Run `git tag` to see a list of all available versions. If you build from `master` you may get [errors](#errors).

Select Build Options:

    ./configure

It might be necessary to run `bazel clean` beforehand, if this isn't the first build with this environment.

Build:

    bazel build --config=opt //tensorflow/tools/pip_package:build_pip_package --local_resources 2048,0.5,1.0 --verbose_failures
    
Create wheel:

    bazel-bin/tensorflow/tools/pip_package/build_pip_package ~/Projects/tf-builds/wheels
    
The wheel can be installed directly with `pip` or can be used to [create a conda package](link).

### Errors

Initially I built with the master branch. I encountered the following error during the build process:

    ModuleNotFoundError: No module named 'keras_applications'

More discussion about the error [here](https://stackoverflow.com/questions/51771039/error-compiling-tensorflow-from-source-no-module-named-keras-applications).
To fix the error, I updated my build environment:

    conda install keras-applications==1.0.4 --no-deps
    conda install keras-preprocessing==1.0.2 --no-deps
    conda install h5py==2.8.0

The build completed successfully after applying the fixes. 
However the new build would give the following error in Keras code that worked previously (this is after calling model.fit_generator() from the stand alone version of Keras):

    `steps_per_epoch=None` is only valid for a generator based on the `keras.utils.Sequence` class. Please specify `steps_per_epoch` or use the `keras.utils.Sequence` class.

If I build with the v1.10.1 branch, I don't get either error.

### Concerns

Almost every completed build that uses jemalloc runs extremely slow (as in at least twice as slow as no jemalloc).
Could there be an issue with my build process?
