---
title: "Singularity: Getting started"
start: true
teaching: 30
exercises: 20
questions:
- "What is Singularity and why might I want to use it?"
objectives:
- "Understand what Singularity is and when you might want to use it."
- "Undertake your first run of a simple Singularity container."
keypoints:
- "Singularity is another container platform and it is often used in cluster/HPC/research environments."
- "Singularity has a different security model to other container platforms, one of the key reasons that it is well suited to HPC and cluster environments."
- "Singularity has its own container image format (SIF)."
- "The `singularity` command can be used to pull images from Singularity Hub and run a container from an image file."
---

The episodes in this lesson will introduce you to the [Singularity](https://sylabs.io/singularity/) container platform and demonstrate how to set up and use Singularity.

This material is split into 2 parts:

*Part I: Basic usage, working with images*
 1. **Singularity: Getting started**: This introductory episode
    
Working with Singularity containers:
<ol start="2">
 <li><strong>The singularity cache: </strong> Why, where and how does Singularity cache images locally?</li>
 <li><strong>Running commands within a Singularity container: </strong> How to run commands within a Singularity container.</li>
 <li><strong>Working with files and Singularity containers: </strong> Moving files into a Singularity container; accessing files on the host from within a container.</li>
 <li><strong>Using Docker images with Singularity: </strong>How to run Singularity containers from Docker images.</li>
 </ol>
 *Part II: Creating images, running parallel codes*
 <ol start="6">
   <li><strong>Preparing to build Singularity images</strong>: Getting started with the Docker Singularity container.</li>
   <li><strong>Building Singularity images</strong>: Explaining how to build and share your own Singularity images.</li>
   <li><strong>Running MPI parallel jobs using Singularity containers</strong>: Explaining how to run MPI parallel codes from within Singularity containers.</li>
</ol>

> ## Work in progress...
> This lesson is new material that is under ongoing development. We will introduce Singularity and demonstrate how to work with it. As the tools and best practices continue to develop, elements of this material are likely to evolve. We welcome any comments or suggestions on how the material can be improved or extended.
{: .callout}

# Singularity - Part I

## What is Singularity?

[Singularity](https://sylabs.io/singularity/) is another container platform. In some ways it appears similar to Docker from a user perspective, but in others, particularly in the system's architecture, it is fundamentally different. These differences mean that Singularity is particularly well-suited to running on distributed, High Performance Computing (HPC) infrastructure.

System administrators will not, generally, install Docker on shared computing platforms such as lab desktops, research clusters or HPC platforms because the design of Docker presents potential security issues for shared platforms with multiple users. Singularity, on the other hand, can be run by end-users entirely within "user space", that is, no special administrative privileges need to be assigned to a user in order for them to run and interact with containers on a platform where Singularity has been installed.

## Getting started with Singularity

### A little history...

Singularity is open source and was initially developed within the research community. Some months ago, the project was "forked" something that is not uncommon within the open source software community, with the software effectively splitting into two projects going in different directions. The fork is being developed by a commercial entity, [Sylabs.io](https://sylabs.io/) who provide both the free, open source [SingularityCE (Community Edition)](https://sylabs.io/singularity) and Pro/Enterprise editions of the software. The original open source Singularity project has recently been [renamed to Apptainer](https://apptainer.org/news/community-announcement-20211130/) and has moved into the Linux Foundation. At the time of writing, Apptainer has recently made their initial release available but this has not yet propagated to many HPC systems. We will generally be working with versions of Singularity released before the fork as part of this course so these changes are not directly relevant. However, it is useful to be aware of this history and that you may see both Singularity and Apptainer being used within the research community over the coming months and years. 

> ## Container technologies on HPC
>
> Singularity/Apptainer are not the only container technologies used on HPC systems - you may also see other container technologies used on HPC platforms you have access to (e.g. Podman, CharlieCloud, Sarus). Singularity/Apptainer are, currently, the most widespread technolgies available on HPC systems. However, many of these technologies work ina similar way for users so what you learn here will help you if you need to use these technologies. Furthermore, all of these technologies provide a way to convert Docker container images to their format as we do with Singularity in this lesson.
>
{: .callout}

Part I of this Singularity material is intended to be undertaken on a remote platform where Singularity has been pre-installed. 

_If you're attending a taught version of this course, you will be provided with access details for a remote platform made available to you for use for Part I of the Singularity material. This platform will have the Singularity software pre-installed._

> ## Installing Singularity on your own laptop/desktop
> If you have a Linux system on which you have administrator access and you would like to install Singularity on this system, some information is provided at the start of [Part II of the Singularity material]({{ page.root }}/06-singularity-images-from-docker/index.html#installing-singularity-on-your-local-system-optional-advanced-task). Unless you are experienced with building software from source code in a Linux environment, we strongly recommend working on a platform with Singularity pre-installed when undertaking this section of the course.
> 
> Later in this material, when building Singularity images, we'll look at running Singularity on your local system through Docker.
{: .callout}

Sign in to the remote platform, with Singularity installed, that you've been provided with access to. Check that the `singularity` command is available in your terminal:

> ## Loading a module
> HPC systems often use *modules* to provide access to software on the system so you may need to use the command:
> ~~~
> $ module load singularity
> ~~~
> {: .language-bash}
> before you can use the `singularity` command on the system. Note: this is not needed on ARCHER2.
{: .callout}

~~~
$ singularity --version
~~~
{: .language-bash}

~~~
singularity version 3.7.3
~~~
{: .output}

Depending on the version of Singularity installed on your system, you may see a different version.

## Images and containers

We'll start with a brief note on the terminology used in this section of the course. We refer to both **_images_** and **_containers_**. What is the distinction between these two terms? 

**_Images_** are bundles of files including an operating system, software and potentially data and other application-related files. They may sometimes be referred to as a _disk image_ or _container image_ and they may be stored in different ways, perhaps as a single file, or as a group of files. Either way, we refer to this file, or collection of files, as an image.

A **_container_** is a virtual environment that is based on an image. That is, the files, applications, tools, etc that are available within a running container are determined by the image that the container is started from. It may be possible to start multiple container instances from an image. You could, perhaps, consider an image to be a form of template from which running container instances can be started.

## Getting an image and running a Singularity container

If you recall from learning about Docker, Docker images are formed of a set of _layers_ that make up the complete image. When you pull a Docker image from Docker Hub, you see the different layers being downloaded to your system. They are stored in your local Docker repository on your system and you can see details of the available images using the `docker` command.

Singularity images are a little different. Singularity uses the [Singularity Image Format (SIF)](https://github.com/sylabs/sif) and images are provided as single `SIF` files (with a `.sif` filename extension). Singularity images can be pulled from [Singularity Hub](https://singularity-hub.org/), a registry for container images. Singularity is also capable of running containers based on images pulled from [Docker Hub](https://hub.docker.com/) and some other sources. We'll look at accessing containers from Docker Hub later in the Singularity material.

> ## Singularity Hub
> [Singularity Hub](https://singularity-hub.org/) is a repository for storing Singularity images. You can browse the stored images by visiting the website. Images can be accessed and run via the `singularity` command.
{: .callout}

Let's begin by creating a `test` directory, changing into it and _pulling_ a test _Hello World_ image from Singularity Hub:

~~~
$ mkdir test
$ cd test
$ singularity pull hello-world.sif shub://vsoch/hello-world
~~~
{: .language-bash}

~~~
INFO:    Downloading shub image
 59.8 MiB / 59.8 MiB [===============================================================================================================] 100.00% 52.03 MiB/s 1s
~~~
{: .output}

What just happened?! We pulled a SIF image from Singularity Hub using the `singularity pull` command and directed it to store the image file using the name `hello-world.sif` in the current directory. If you run the `ls` command, you should see that the `hello-world.sif` file is now present in the current directory. This is our image and we can now run a container based on this image:

~~~
$ singularity run hello-world.sif
~~~
{: .language-bash}

~~~
RaawwWWWWWRRRR!! Avocado!
~~~
{: .output}

The above command ran a _singularity container_ from the `hello-world.sif` image that we downloaded from Singularity Hub and the resulting output was shown. 


How did the container determine what to do when we ran it?! What did running the container actually do to result in the displayed output?

When you run a container from a Singularity image without using any additional command line arguments, the container runs the default run script that is embedded within the image. This is a shell script that can be used to run commands, tools or applications stored within the image on container startup. We can inspect the image's run script using the `singularity inspect` command:

~~~
$ singularity inspect -r hello-world.sif
~~~
{: .language-bash}

~~~
#!/bin/sh 

exec /bin/bash /rawr.sh

~~~
{: .output}

This shows us the script within the `hello-world.sif` image configured to run by default when we use the `singularity run` command.

That concludes this introductory Singularity episode. The next episode looks in more detail at running containers.
