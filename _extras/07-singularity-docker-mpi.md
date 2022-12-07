---
title: "Additional material: Running MPI parallel jobs using Singularity containers"
teaching: 30
exercises: 40
questions:
- "How do I set up and run an MPI job from a Singularity container?"
objectives:
- "Learn how MPI applications within Singularity containers can be run on HPC platforms"
- "Understand the challenges and related performance implications when running MPI jobs via Singularity"
keypoints:
- "Singularity images containing MPI applications can be built on one platform and then run on another (e.g. an HPC cluster) if the two platforms have compatible MPI implementations."
- "When running an MPI application within a Singularity container, use the MPI executable on the host system to launch a Singularity container for each process."
- "Think about parallel application performance requirements and how where you build/run your image may affect that."
---

## MPI overview

MPI - [Message Passing Interface](https://en.wikipedia.org/wiki/Message_Passing_Interface) - is a widely used standard for parallel programming. It is used for exchanging messages/data between processes in a parallel application. If you've been involved in developing or working with computational science software, you may already be familiar with MPI and running MPI applications.

When working with an MPI code on a large-scale cluster, a common approach is to compile the code yourself, within your own user directory on the cluster platform, building against the supported MPI implementation on the cluster. Alternatively, if the code is widely used on the cluster, the platform administrators may build and package the application as a module so that it is easily accessible by all users of the cluster.

## MPI codes with Singularity containers

We've already seen that building Singularity containers can be impractical without root access. Since we're highly unlikely to have root access on a large institutional, regional or national cluster, building a container directly on the target platform is not normally an option.

If our target platform uses [OpenMPI](https://www.open-mpi.org/), one of the two widely used source MPI implementations, we can build/install a compatible OpenMPI version on our local build platform, or directly within the image as part of the image build process. We can then build our code that requires MPI, either interactively in an image sandbox or via a definition file.

If the target platform uses a version of MPI based on [MPICH](https://www.mpich.org/), the other widely used open source MPI implementation, there is [ABI compatibility between MPICH and several other MPI implementations](https://www.mpich.org/abi/). In this case, you can build MPICH and your code on a local platform, within an image sandbox or as part of the image build process via a definition file, and you should be able to successfully run containers based on this image on your target cluster platform.

As described in Singularity's [MPI documentation](https://apptainer.org/user-docs/3.7/mpi.html), support for both OpenMPI and MPICH is provided. Instructions are given for building the relevant MPI version from source via a definition file. We will use a modified version of this to build a container image using Docker below.

### **Container portability and performance on HPC platforms**

While building a container on a local system that is intended for use on a remote HPC platform does provide some level of portability, if you're after the best possible performance, it can present some issues. The version of MPI in the container will need to be built and configured to support the hardware on your target platform if the best possible performance is to be achieved. Where a platform has specialist hardware with proprietary drivers, building on a different platform with different hardware present means that building with the right driver support for optimal performance is not likely to be possible. This is especially true if the version of MPI available is different (but compatible). Singularity's [MPI documentation](https://apptainer.org/user-docs/3.7/mpi.html) highlights two different models for working with MPI codes. The _[hybrid model](https://apptainer.org/user-docs/3.7/mpi.html#hybrid-model)_ that we'll be looking at here involves using the MPI executable from the MPI installation on the host system to launch singularity and run the application within the container. The application in the container is linked against and uses the MPI installation within the container which, in turn, communicates with the MPI daemon process running on the host system. In the following section we'll look at building a Singularity image containing a small MPI application that can then be run using the hybrid model.

## Building and running a Singularity image for an MPI code

### Building and testing an image

This example makes the assumption that you'll be building a container image on a local platform and then deploying it to a cluster with a different but compatible MPI implementation. See [Singularity and MPI applications](https://apptainer.org/user-docs/3.7/mpi.html#singularity-and-mpi-applications) in the Singularity documentation for further information on how this works.

We'll build an image from a definition file. Containers based on this image will be able to run MPI benchmarks using the [OSU Micro-Benchmarks](https://mvapich.cse.ohio-state.edu/benchmarks/) software.

In this example, the target platform is the ARCHER2 supercomputer (a HPE Cray EX system with 750,080 CPU cores on 5,860 compute nodes). ARCHER2 uses HPE Cray MPICH as the MPI library - a derivative of the open source MPICH MPI distribution modified for use with the HPE Cray Slingshot interconnect on ARCHER2.

Begin by creating a `osu-benchmarks` directory on your local system:

```
mkdir osu-benchmarks
cd osu-benchmarks
```

In the same directory, create a Dockerfile that looks like:

```
FROM ubuntu:20.04

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get -y update && apt-get -y install curl build-essential libfabric-dev libibverbs-dev gfortran

RUN curl -sSLO http://www.mpich.org/static/downloads/3.4.2/mpich-3.4.2.tar.gz \
   && tar -xzf mpich-3.4.2.tar.gz -C /root \
   && cd /root/mpich-3.4.2 \
   && ./configure --prefix=/usr --with-device=ch4:ofi --disable-fortran \
   && make -j8 install \
   && rm -rf /root/mpich-3.4.2 \
   && rm /mpich-3.4.2.tar.gz

RUN curl -sSLO http://mvapich.cse.ohio-state.edu/download/mvapich/osu-micro-benchmarks-5.4.1.tar.gz \
   && tar -xzf osu-micro-benchmarks-5.4.1.tar.gz -C /root \
   && cd /root/osu-micro-benchmarks-5.4.1 \
   && ./configure --prefix=/usr/local CC=/usr/bin/mpicc CXX=/usr/bin/mpicxx \
   && cd mpi \
   && make -j8 install \
   && rm -rf /root/osu-micro-benchmarks-5.4.1 \
   && rm /osu-micro-benchmarks-5.4.1.tar.gz

ENV PATH /usr/local/libexec/osu-micro-benchmarks/mpi/pt2pt:$PATH
ENV PATH /usr/local/libexec/osu-micro-benchmarks/mpi/collective:$PATH

CMD osu_latency
```
{: .output}

A quick overview of what the above Dockerfile is doing:

 - The image is built starting from the `ubuntu:20.04` Docker image.
 - `ENV` commands: Set a couple of environment variables that will be available within all containers run from the generated image.
 - `RUN` commands:
   + Ubuntu's `apt-get` package manager is used to update the package directory and then install the compilers and other libraries required for the MPICH build.
   + The MPICH distribution .tar.gz file is downloaded, extracted and the configure, build and install steps are run. Note the use of the `--with-device` option to configure MPICH to use the CH4 device to support improved communication performance on a high performance cluster that provides support for this.
   + The OSU Micro-Benchmarks .tgz file is downloaded, extracted and the configure, build and install steps are run to build the benchmark code from source.
 - `CMD`: Sets the default action for the container - runs the `osu_latency` benchmark.

_Note that base path of the the executable to run (`$OSU_DIR`) is hardcoded in the run script_. The command line parameter that you provide when running a container instance based on the image is then added to this base path. Example command line parameters include: `startup/osu_hello`, `collective/osu_allgather`, `pt2pt/osu_latency`, `one-sided/osu_put_latency`.

> ## Build and test the OSU Micro-Benchmarks image
>
> Using the above Dockerfile, build a Docker container image for the `linux/amd64` platform
> named `<your-dockerhub-id>/osu-benchmarks`.
> 
> Once the image has finished building, push it to Dockerhub.
> 
> > ## Solution
> > 
> > You should be able to build a Docker container image as follows (for a Dockerhub username `alice`):
> > 
> > ~~~
> > docker image build --platform linux/amd64 -t alice/osu-benchmarks .
> > ~~~
> > {: .language-bash}
> >
> > ~~~
> >[+] Building 3908.4s (9/9) FINISHED                                                 
> > => [internal] load build definition from Dockerfile                                               0.0s
> > => => transferring dockerfile: 1.06kB                                      0.0s
> > => [internal] load .dockerignore                    0.0s
> > => => transferring context: 2B                       0.0s
> > => [internal] load metadata for docker.io/library/ubuntu:20.04                1.1s
> > => [auth] library/ubuntu:pull token for registry-1.docker.io                      0.0s
> > => CACHED [1/4] FROM docker.io/library/ubuntu:20.04@sha256:450e066588f42ebe1551f3b1a535034b6aa46cd936fe7f2c6b0d72997ec61dbd    0.0s
> > => [2/4] RUN apt-get -y update && apt-get -y install curl build-essential libfabric-dev libibverbs-dev gfortran               152.4s
> > => [3/4] RUN curl -sSLO http://www.mpich.org/static/downloads/3.4.2/mpich-3.4.2.tar.gz    && tar -xzf mpich-3.4.2.tar.gz -C /
root    && cd /root/mpich-3.4.2     3639.3s
> > => [4/4] RUN curl -sSLO http://mvapich.cse.ohio-state.edu/download/mvapich/osu-micro-benchmarks-5.4.1.tar.gz  && tar -xzf osu-micro-benchmarks-5.4.1.tar.gz -C  114.6s 
> > => exporting to image         0.9s 
> > => => exporting layers           0.9s 
> > => => writing image sha256:a649a8826b6e47b1ae1835bea3a8148bf67bda75ed0fb62bc3412dd361d43871           0.0s 
> > => => naming to docker.io/aturnerepcc/osu-benchmarks    
> > ~~~
> > {: .output}
> >
> > Note: this will take a long time to build (sometimes over 1 hour)! (We have a prebuilt version for you to use on ARCHER2 so you
> > can carry on with the course without waiting for the build process to complete.)
> >
> > Once it has built, you can push to Dockerhub with (again for Dockerhub username `alice`):
> > 
> > ~~~
> > docker push alice/osu-benchmarks
> > ~~~
> > {: .language-bash}
> {: .solution}
{: .challenge}

## Running Singularity containers via MPI

We can now try undertaking a parallel run of one of the OSU benchmarking tools within our container image.

This is where things get interesting and we'll begin by looking at how Singularity containers are run within an MPI environment.

If you're familiar with running MPI codes, you'll know that you use `mpirun`, `mpiexec`, `srun` or a similar MPI executable to start your application. This executable may be run directly on the local system or cluster platform that you're using, or you may need to run it through a job script submitted to a job scheduler. Your MPI-based application code, which will be linked against the MPI libraries, will make MPI API calls into these MPI libraries which in turn talk to the MPI daemon process running on the host system. This daemon process handles the communication between MPI processes, including talking to the daemons on other nodes to exchange information between processes running on different machines, as necessary.

When running code within a Singularity container, we don't use the MPI executables stored within the container (i.e. we DO NOT run `singularity exec mpirun -np <numprocs> /path/to/my/executable`). Instead we use the MPI installation on the host system to run Singularity and start an instance of our executable from within a container for each MPI process. Without Singularity support in an MPI implementation, this results in starting a separate Singularity container instance within each process. This can present some overhead if a large number of processes are being run on a host. Where Singularity support is built into an MPI implementation this can address this potential issue and reduce the overhead of running code from within a container as part of an MPI job.

Ultimately, this means that our running MPI code is linking to the MPI libraries from the MPI install within our container and these are, in turn, communicating with the MPI daemon on the host system which is part of the host system's MPI installation. In the case of MPICH, these two installations of MPI may be different but as long as there is [ABI compatibility](https://wiki.mpich.org/mpich/index.php/ABI_Compatibility_Initiative) between the version of MPI installed in your container image and the version on the host system, your job should run successfully.

We can now try running a 2-process MPI run of a point to point benchmark `osu_latency`.

## Undertake a parallel run of the `osu_latency` benchmark

_[ARCHER2](https://www.archer2.ac.uk/), the UK National Supercomputing Service, uses the [Slurm workload manager](https://www.schedmd.com/) to manage the submission and running of jobs. We provide you with a template Slurm job submission script in this section for running a parallel job via your Singularity container on ARCHER2.

_This version of the exercise, for undertaking a parallel run of the osu_latency benchmark with your Singularity container that contains an MPI build, is specific to this run of the course._

Log into ARCHER2 and move to your work file space:

```
remote> cd /work/ta089/ta089/$USER
```
{: .language-bash}

Pull the `osu-benchmarks` contianer image from Dockerhub. If your build process finished in time, you can substitute the `aturnerepcc` Dockerhub ID for your own Dockerhub ID.

```
remote> singularity pull osu-benchmarks.sif docker://aturnerepcc/osu-benchmarks
```
{: .language-bash}
```
INFO:    Converting OCI blobs to SIF format
INFO:    Starting build...
Getting image source signatures
Copying blob eaead16dc43b skipped: already exists  
Copying blob 4080d37551cf done  
Copying blob 1e049ea0e2a4 done  
Copying blob 3cfa71560203 done  
Copying config 2ee4e1caa7 done  
Writing manifest to image destination
Storing signatures
2022/12/07 20:47:52  info unpack layer: sha256:eaead16dc43bb8811d4ff450935d607f9ba4baffda4fc110cc402fa43f601d83
2022/12/07 20:47:52  warn xattr{etc/gshadow} ignoring ENOTSUP on setxattr "user.rootlesscontainers"
2022/12/07 20:47:52  warn xattr{/tmp/build-temp-489555399/rootfs/etc/gshadow} destination filesystem does not support xattrs, further warnings will be suppressed
2022/12/07 20:47:53  info unpack layer: sha256:4080d37551cf3ae041eee7c1915aac5c8a034fe4f31a5bf7cbbbbc6c34d25415
2022/12/07 20:47:53  warn xattr{etc/gshadow} ignoring ENOTSUP on setxattr "user.rootlesscontainers"
2022/12/07 20:47:53  warn xattr{/tmp/build-temp-489555399/rootfs/etc/gshadow} destination filesystem does not support xattrs, further warnings will be suppressed
2022/12/07 20:47:56  info unpack layer: sha256:1e049ea0e2a4b7cd0213ace5976336f19748288926768f5d2002118f8641ddea
2022/12/07 20:47:57  info unpack layer: sha256:3cfa71560203aa648cee53b4767aeab39b9fbf2bff3bb356c098368c6f0dc91e
INFO:    Creating SIF file...
```
{: .output}

It is now necessary to create a `Slurm` job submission script to run the benchmark example.

Download this [template script]({{site.url}}{{site.baseurl}}/files/osu_latency.slurm.template) on ARCHER2 and edit it to suit your configuration. (_Copy the link and use a tool such as `wget` to download the template._)

Submit the job submission script to the `Slurm` scheduler using the `sbatch` command.

~~~
remote> sbatch osu_latency.slurm
~~~
{: .language-bash}

## Expected output and discussion

As you will have seen in the commands using the provided template job submission script, we have called `srun` on the host system and are passing to MPI the `singularity` executable for which the parameters are the image file and any parameters we want to pass to the image's run script. In this case, the parameter is the name of the benchmark executable to run - `osu_latency`.

The following shows an example of the output you should expect to see. You should have latency values shown for message sizes up to 4MB.

~~~
# OSU MPI Latency Test v5.8
# Size          Latency (us)
0                       2.23
1                       2.22
2                       2.22
...
4194304               354.06
 ~~~
{: .output}

This has demonstrated that we can successfully run a parallel MPI executable from within a Singularity container. However, depending on the configuration of the target cluster platform, it's possible that the two processes will have run on the same physical node - if so, this is not testing the performance of the interconnects between nodes.

You could now try running a larger-scale test. You can also try running a benchmark that uses multiple processes, for example try `osu_allreduce`.

> ## [Advanced] Investigate performance when using a container image built on a local system and run on a cluster
> 
> _This is an advanced exercise, we will not cover it during the taught version of the course but provide it as something you could try if you're interested to investigate potential performance differences between different approaches to building and running MPI code._
>
> To get an idea of any difference in performance between the code within your Singularity image and the same code built natively on the target HPC platform, try building the OSU benchmarks from source, locally on the cluster. Then try running the same benchmark(s) that you ran via the singularity container. Have a look at the outputs you get when running `osu_allreduce` or one of the other collective benchmarks to get an idea of whether there is a performance difference and how significant it is.
> 
> Try running with enough processes that the processes are spread across different physical nodes so that you're making use of the cluster's network interconnects.
> 
> What do you see?
> 
> > ## Discussion
> > You may find that performance is significantly better with the version of the code built directly on the HPC platform. Alternatively, performance may be similar between the two versions.
> > 
> > How big is the performance difference between the two builds of the code?
> > 
> > What might account for any difference in performance between the two builds of the code?
> > 
> {: .solution}
{: .challenge}

If performance is an issue for you with codes that you'd like to run via Singularity, you are advised to take a look at using the _[bind model](https://apptainer.org/user-docs/3.7/mpi.html#bind-model)_ for building/running MPI applications through Singularity.
