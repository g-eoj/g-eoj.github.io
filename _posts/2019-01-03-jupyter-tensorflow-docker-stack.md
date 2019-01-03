---
layout: post
title: Custom Jupyter/TensorFlow Docker Stack
tags: [docker, tensorflow, jupyter, notebook]
---

This post documents setting up a Jupyter/TensorFlow Docker stack that performs well on an old Core2Duo CPU and Windows 7.
<!--more-->
I do a lot of projects with Jupyter notebooks and TensorFlow on this system&mdash;I've discovered that:
- TensorFlow is quicker on a Virtual Box Linux guest system.
- TensorFlow with Eigen is quicker than TensorFlow with MKL, which is the default math library.

Altogether, TensorFlow with Eigen on Linux trains convolutional neural networks 4x faster than default TensorFlow on Windows 7. 
However, using a full Linux distro in Virtual Box eats a significant chunk of ram.
The steps outlined in this post use [Docker Toolbox](https://docs.docker.com/toolbox/overview/) and [Jupyter Docker Stacks](https://jupyter-docker-stacks.readthedocs.io/en/latest/) for a memory efficient solution. Additionally, it allows me to work on files stored locally on the Windows 7 host.

#### Steps Outline
- [Setup Docker Toolbox](#setup-docker-toolbox)
	- [Change .docker Directory (optional)](#change-docker-directory-optional)
	- [Setup Port Forwarding](#setup-port-forwarding)
	- [Setup Shared Folder](#setup-shared-folder)
- [Make Custom Docker Image](#make-custom-docker-image)
	- [Dockerfile](#dockerfile)
	- [Build Image](#build-image)
- [Usage](#usage)


## Setup Docker Toolbox
[Docker Toolbox](https://docs.docker.com/toolbox/overview/) is the only option for running Docker on Windows 7. It uses Virtual Box for the Docker virtual machine.

Besides just running the Docker Toolbox installer, do the following steps:

### Change .docker Directory (optional)

The Docker Virtual Box image can get quite big and you may not want to store it in the default install location. The easiest solution I found involves moving the `.docker` folder to the desired location and then creating a junction[^1]:

	$ C:\Users\username>mklink /j .docker D:\.docker
	Junction created for .docker <<===>> D:\.docker	

The image can also be resized by deleting it and creating a new one with a different size[^2]:

	$ docker-machine rm default
	$ docker-machine create -d virtualbox --virtualbox-disk-size "20000" default
	
### Setup Port Forwarding
To access the Jupyter Notebook server with a local web browser, you need to use port forwarding for the Docker virtual machine in Virtual Box:

![Port Forwarding](/assets/2019-01-03-jupyter-tensorflow-docker-stack/vbox-port-forward.PNG)

### Setup Shared Folder
To work on local files, setup a shared folder for the local directory you work from:

![Shared Folder](/assets/2019-01-03-jupyter-tensorflow-docker-stack/vbox-shared-folder.PNG)
	
## Make Custom Docker Image

The [Jupyter Docker Stacks](https://jupyter-docker-stacks.readthedocs.io/en/latest/) are a collection of pre-made Docker images for working with Jupyter notebooks. 
For this project, the [Deep Learning Stack](https://github.com/jupyter/docker-stacks/tree/master/tensorflow-notebook) is close to what I want, so I use it as a starting point.


### Dockerfile

Below is an edited Dockerfile from the [Deep Learning Stack](https://github.com/jupyter/docker-stacks/tree/master/tensorflow-notebook) that does the following:

- Installs `tensorflow-eigen` instead of `tensorflow`.
- Adds `custom.css` to the Docker image for Jupyter notebook styling.
- Installs `jupyter_contrib_nbextensions` and enables the spell checker by default.

The edited Dockerfile looks like this:

	# Copyright (c) Jupyter Development Team.
	# Distributed under the terms of the Modified BSD License.
	ARG BASE_CONTAINER=jupyter/scipy-notebook
	FROM $BASE_CONTAINER

	# Install Tensorflow
	RUN conda install --quiet --yes \
		'tensorflow-eigen=1.12*' \
		'keras=2.2*' && \
		conda clean -tipsy && \	
		fix-permissions $CONDA_DIR && \
		fix-permissions /home/$NB_USER
		
	COPY custom.css /home/$NB_USER/.jupyter/custom/

	# Install nbextensions
	RUN conda install --quiet --yes -c 'conda-forge' \
		'jupyter_contrib_nbextensions=0.5*' && \
		conda clean -tipsy && \	
		fix-permissions $CONDA_DIR && \
		fix-permissions /home/$NB_USER && \
		jupyter nbextension enable spellchecker/main
	
### Build Image
Once the files are created, place them in a folder like so:

	tensorflow-eigen-notebook
	├── custom.css
	└── Dockerfile
	
Opening the Docker Quick Start Terminal will start the Docker virtual machine. In the terminal, navigate into the `tensorflow-eigen-notebook` directory and run:

	docker build -t `jupyter/tensorflow-eigen-notebook` .

## Usage	

To start the notebook server on a local directory, first use the Docker Quick Start Terminal (or start Docker some other way). Navigate to the directory containing the files you want to work on, then run:

	docker run --rm -p 8888:8888 -v "$PWD":/home/jovyan/work jupyter/tensorflow-eigen-notebook

Next, start a web browser and go to `127.0.0.1:8888` to access the notebook tree.
When you want to shut the notebook server down, just select quit from the web UI.

#### Useful Commands

When you're finished working and want to free up resources, remember to stop the Docker virtual machine from the terminal:

	docker-machine stop

You may want to adjust how much memory is available to the Docker virtual machine in Virtual Box based on how much memory Docker needs. To see how much memory and other resources Docker is using, run:

	docker stats	


[^1]: [Change Docker machine location - Windows](https://stackoverflow.com/questions/33933107/change-docker-machine-location-windows#37246965)
[^2]: [Increase Disk Space on Docker Toolbox](https://stackoverflow.com/questions/39811650/increase-disk-space-on-docker-toolbox)