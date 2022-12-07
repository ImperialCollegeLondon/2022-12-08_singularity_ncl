---
title: "Preparing to build Singularity images"
teaching: 15
exercises: 20
questions:
- "What environment do I need to build a container image to use with Singularity?"
objectives:
- "Recap building Docker container images and pushing to Dockerhub."
- "Demonstrate pulling the container image from Dockerhub onto the remote system using Singularity."
keypoints:
- "You can build container images using Docker and run them using Singularity."
- "You must build the Docker container image with the correct processor architecture to match your remote HPC system."
- "Dockerhub makes it easier to transfer container images from your local system to a remote HPC system."
---

# Singularity - Part II

## Brief recap

In the five episodes covering Part I of this Singularity material we've seen how Singularity can be used on a computing platform where you don't have any administrative privileges. The software was pre-installed and it was possible to work with existing images such as Singularity image files already stored on the platform or images obtained from a remote image repository such as Singularity Hub or Docker Hub.

It is clear that between Singularity Hub and Docker Hub there is a huge array of images available, pre-configured with a wide range of software applications, tools and services. But what if you want to create your own images or customise existing images?

In this first of three episodes in Part II of the Singularity material, we'll recap on how to build Docker images and push them to Dockerhub so they can be downloaded for use with Singularity on the remote system.

## Building Docker container images - recap

The key things needed to build a Docker image to push to Dockerhub are:

 - A working Docker installation(!)
 - A `Dockerfile`
 - Any other files you wish to include in the container image
 - A Dockerhub account

We will build a simple container containing a Python 3 installation based on Ubuntu

Create a working directory for this image on your local host system where you have administrator privileges and
move into it:

```
mkdir ubuntu-python
cd ubuntu-python
```
{: .language-bash}

Next create s simple `Dockerfile` that starts from Ubuntu, installs Python 3 and, by default, prints the
Python version:

```
FROM ubuntu:20.04
RUN apt-get -y update
RUN apt-get install -y python3
CMD ["python3", "--version"]
```
{: .language-bash}

Use Docker to build the image and make sure it is compatible with the HPC platform by specifying the
x86_64 architecture:

```
docker image build -t alice/ubuntu-python --platform linux/amd64 .
```
{: .language-bash}
```
[+] Building 76.3s (8/8) FINISHED
 => [internal] load build definition from Dockerfile                                               0.0s
 => => transferring dockerfile: 137B                                                               0.0s
 => [internal] load .dockerignore                                                                  0.0s
 => => transferring context: 2B                                                                    0.0s
 => [internal] load metadata for docker.io/library/ubuntu:20.04                                    1.9s
 => [auth] library/ubuntu:pull token for registry-1.docker.io                                      0.0s
 => [1/3] FROM docker.io/library/ubuntu:20.04@sha256:450e066588f42ebe1551f3b1a535034b6aa46cd936fe7f2c6b0d72997ec61dbd  22.6s
 => => resolve docker.io/library/ubuntu:20.04@sha256:450e066588f42ebe1551f3b1a535034b6aa46cd936fe7f2c6b0d72997ec61dbd                        0.0s
 => => sha256:450e066588f42ebe1551f3b1a535034b6aa46cd936fe7f2c6b0d72997ec61dbd 1.42kB / 1.42kB     0.0s
 => => sha256:b25ef49a40b7797937d0d23eca3b0a41701af6757afca23d504d50826f0b37ce 529B / 529B         0.0s
 => => sha256:680e5dfb52c74a1fbc99c2922c8e25b5736e6cd1a3d9430890d52a4f8f44087a 1.46kB / 1.46kB     0.0s
 => => sha256:eaead16dc43bb8811d4ff450935d607f9ba4baffda4fc110cc402fa43f601d83 28.58MB / 28.58MB  21.7s
 => => extracting sha256:eaead16dc43bb8811d4ff450935d607f9ba4baffda4fc110cc402fa43f601d83          0.7s
 => [2/3] RUN apt-get -y update                                                                   27.7s
 => [3/3] RUN apt-get install -y python3                                                          23.9s
 => exporting to image                                                                             0.1s 
 => => exporting layers                                                                            0.1s 
 => => writing image sha256:c58fd754f0e5540b4a9c7768816eea8dcf742bd7bb06a38a37f06390ddc035cb       0.0s 
 => => naming to docker.io/aturnerepcc/ubuntu-python   
```
{; .output}

Note: if we are on an x86_64 platform (Intel or AMD processor) then the `--platform linux/amd64` flag is not strictly 
needed in this case. You should check what processor architecture your HPC system has to choose the right
flag for building container images.

Push to Dockerhub:

```
docker push alice/ubuntu-python
```
{: .language-bash}
```
Using default tag: latest
The push refers to repository [docker.io/aturnerepcc/ubuntu-python]
cfd62b5df445: Pushed 
4a912cbded83: Pushed 
f4462d5b2da2: Mounted from library/ubuntu 
latest: digest: sha256:17823c31c6b86f117bf24df4e19a39077ba36a9c6e45010b0a4853de789a245a size: 953
```
{; .output}

We should now have a Docker container image hosted on Dockerhub with the correct architecture for
our HPC system. We will now log into the HPC system and pull and run the image using Singularity:

```
remote> singularity pull docker://alice/ubuntu-python
```
{: .language-bash}
```
INFO:    Converting OCI blobs to SIF format
INFO:    Starting build...
Getting image source signatures
Copying blob eaead16dc43b done  
Copying blob ba18c8174437 done  
Copying blob c3ad949686c8 done  
Copying config e395a30cf4 done  
Writing manifest to image destination
Storing signatures
2022/12/07 19:20:02  info unpack layer: sha256:eaead16dc43bb8811d4ff450935d607f9ba4baffda4fc110cc402fa43f601d83
2022/12/07 19:20:02  warn xattr{etc/gshadow} ignoring ENOTSUP on setxattr "user.rootlesscontainers"
2022/12/07 19:20:02  warn xattr{/tmp/build-temp-818601781/rootfs/etc/gshadow} destination filesystem does not support xattrs, further warnings will be suppressed
2022/12/07 19:20:03  info unpack layer: sha256:ba18c8174437bd8247c0b264b8fca14e42ff8eddee632c444ed5da6440432e07
2022/12/07 19:20:03  warn xattr{var/lib/apt/lists/auxfiles} ignoring ENOTSUP on setxattr "user.rootlesscontainers"
2022/12/07 19:20:03  warn xattr{/tmp/build-temp-818601781/rootfs/var/lib/apt/lists/auxfiles} destination filesystem does not support xattrs, further warnings will be suppressed
2022/12/07 19:20:03  info unpack layer: sha256:c3ad949686c81069aaf91b0ec53b6110109e8172e4d067cb2f50b1f155ae5b7c
2022/12/07 19:20:04  warn xattr{usr/local/lib/python3.8} ignoring ENOTSUP on setxattr "user.rootlesscontainers"
2022/12/07 19:20:04  warn xattr{/tmp/build-temp-818601781/rootfs/usr/local/lib/python3.8} destination filesystem does not support xattrs, further warnings will be suppressed
INFO:    Creating SIF file...
```
{; .output}

```
remote> singularity run ubuntu-python_latest.sif
```
{: .language-bash}
```
Python 3.8.10
```
{; .output}

> ## Cluster platform configuration for running Singularity containers
>
> For testing purposes, we can run our image that we've created on ARCHER2 by copying the image to the platform and running it interactively at the terminal.
>
> However, for a real job, this is not an option and we would be required to submit our job to the job scheduler on ARCHER2 to have it run on the system's compute nodes.
>
> Doing this requires a number of configuration parameters to be specified. The parameters that need to be set and the values that they need to be set to are specific to the cluster you're running on. In the next section, where we look at running parallel jobs, we'll provide platform-specfic parameters for ARCHER2. However, if you're running on your own institutional cluster or another HPC platform that you have access to, it is likely that you'll need to liaise with the system administrators or look at documentation to identify the parameters that need to be passed to Singularity.
> 
{: .callout}

## Additional Singularity features

Singularity has a wide range of features. You can find full details in the [Singularity User Guide](https://apptainer.org/user-docs/3.7/index.html) and we highlight a couple of key features here that may be of use/interest:

**Signing containers:** If you do want to share container image (`.sif`) files directly with colleagues or collaborators, how can the people you send an image to be sure that they have received the file without it being tampered with or suffering from corruption during transfer/storage? And how can you be sure that the same goes for any container image file you receive from others? Singularity supports signing containers. This allows a digital signature to be linked to an image file. This signature can be used to verify that an image file has been signed by the holder of a specific key and that the file is unchanged from when it was signed. You can find full details of how to use this functionality in the Singularity documentation on [Signing and Verifying Containers](https://apptainer.org/user-docs/3.7/signNverify.html).

> ## Installing Singularity on your local system (optional) \[Advanced task\]
>
> If you are running Linux and would like to install Singularity locally on your system, the source code is provided via the [Apptainer project](https://apptainer.org)'s [Singularity repository](https://github.com/apptainer/singularity). See the releases [here](https://github.com/apptainer/singularity/releases/). You will need to install various dependencies on your system and then build Singularity from source code.
>
> _If you are not familiar with building applications from source code in a Linux environment, it is strongly recommended that you use  Singularity installed on a remote HPC resource rather than attempting to build and install Singularity yourself. The installation process is an advanced task that is beyond the scope of this session._
> 
> However, if you have Linux systems knowledge and would like to attempt a local install of Singularity, you can find details in the [INSTALL.md](https://github.com/apptainer/singularity/blob/master/INSTALL.md) file within the Singularity repository that explains how to install the prerequisites and build and install the software. Singularity is written in the [Go](https://golang.org/) programming language and Go is the main dependency that you'll need to install on your system. The process of installing Go and any other requirements is detailed in the INSTALL.md file.
> 
{: .callout}

> ## Note
> If you do not have access to a system with Docker installed, or a Linux system where you can build and install Singularity but you have administrative privileges on another system, you could look at installing a virtualisation tool such as [VirtualBox](https://www.virtualbox.org/) on which you could run a Linux Virtual Machine (VM) image. Within the Linux VM image, you will be able to install Singularity. Again this is beyond the scope of the course.
>
> If you are not able to access/run Singularity yourself on a system where you have administrative privileges, you can still follow through this material as it is being taught (or read through it in your own time if you're not participating in a taught version of the course) since it will be helpful to have an understanding of how Singularity images can be built.
> 
> You could also attempt to follow this section of the lesson without using root and instead using the `singularity` command's [`--fakeroot`](https://apptainer.org/user-docs/3.7/fakeroot.html) option. However, you may encounter issues with permissions when trying to build images and run your containers and this is why running the commands as root is strongly recommended and is the approach described in this lesson.
{: .callout}

