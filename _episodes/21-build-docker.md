---
title: "Building images with Docker"
teaching: 15
exercises: 5
questions:
objectives:
- Discuss the pros & cons of building with Singularity vs Docker
- Learn the basic syntax of a Dockerfile
- Build and share an image with Docker
- Convert a Docker image into the Singularity format
keypoints:
- Use `FROM` to specify the build starting image
- Execute shell commands with `RUN`
- Declare environment variables with `ENV`
- Build images starting from a `Dockerfile` recipe with `docker build`
- Push images to a web registry with `docker push`
- Share images as a single file using `docker save` and `docker load`
- You can locally convert a Docker image by using `singularity pull docker-daemon:`
---


### Why using Docker for container builds?

Over the entirity of this tutorial, we're proposing Singularity as the principal tool to handle and run containerised applications in HPC.  Do we really need to extend our toolkit to include [Docker](https://hub.docker.com/search/?type=edition&offering=community)?

At present, both tools, Docker and Singularity (and its successor Apptainer), require root privileges for building, implying that this step cannot be performed on HPC systems. Both require access to a machine where a user has administrative privileges. However, each tool has a different build process. To better inform an answer to the question of Docker's utility, here are some of the advantages when building with one or the other tool.

> ## Advantages of each container engine
> #### Docker
> * Image format can be run by all existing container engines, including Singularity.
> * Docker uses _layered_ image format compliant with the *Open Containers Initiative* (OCI) allows caching, for reduced build time during prototyping
> and development.
>   * Each build command in the recipe file results in the creation of a distinct image layer,
> which are cached during the build process. Repeated build attempts that use the same layers will exploit the cache
> reducing the overall build time and aiding debugging.
>   * A drawback to this approach is shipping a container image is not straightforward
> and requires either relying on a public registry, or compressing the image in a *tar* archive.
>
> #### Singularity
> * Unambiguous container usage modes, via distinct keywords: `exec`, `shell`, `run`, `instance` (see episode on web applications).
> * Powerful ways of defining shell environment (see [Singularity docs on Environment and Metadata](https://singularity.hpcng.org/user-docs/3.5/environment_and_metadata.html));
> * Ability to sign/verify images for improved security.
> * *Singularity Image Format* (SIF) is a single file layout for container images that can be handled as any other file and shared easily.
>   * Building on this single file format, a number of features have been developed, such as image signing and verification, and (more recently) image encryption.  
>   * A drawback of this approach is that during build time a progressive, incremental approach is not possible as it is with Docker.
{: .discussion}

Docker is preferred in most situations due to its ability to layered format and critically, its **compatibility**.  Build with Docker, and you'll know the resulting image can be run from every container engine anywhere in the world.

### Building the container image with Docker

Let's `cd` into the relevant demo directory:

```bash
$ cd $TUTO/demos/lolcow_docker
```
{: .source}

Let us also start building the image with `docker build` straight away while we progress
through the material.

```bash
$ sudo docker build -t lolcow:1Nov19 .
```
{: .source}

In the command above, `.` is the location of the build context (*i.e.* the directory for the Dockerfile).  
The `-t` flag is used to specify the image name (compulsory) and tag (optional).

Any lowercase alphanumeric string can be used as image name; here we've used `lolcow`.  The image tag (following the colon) can be optionally used to maintain a set of different image versions on Docker Hub, and is a key feature in enabling reproducibility of your computations through containers; here we've used `1Nov19`.

Adding the prefix `<Your Docker Hub account>/` to the image name is also optional and allows to push the built image to your Docker Hub registry (see below).

The complete format for the image name looks like: `<Your Docker Hub account ^>/<Image name>:<Image tag ^>`. `^`These are optional.

This is the output of our build:

```
Sending build context to Docker daemon  2.048kB
Step 1/7 : FROM ubuntu:18.04
 ---> 775349758637
Step 2/7 : LABEL maintainer="Pawsey Supercomputing Centre"
 ---> Running in 91c109dfd5ba
Removing intermediate container 91c109dfd5ba
 ---> 361490204a2c
Step 3/7 : RUN apt-get -y update &&   apt-get -y install fortune cowsay lolcat
 ---> Running in 4543b6bb99f1

[..]

Removing intermediate container 4543b6bb99f1
 ---> 7958a569068f
Step 4/7 : ENV PATH=/usr/games:$PATH
 ---> Running in 86282799c41f
Removing intermediate container 86282799c41f
 ---> 3ffdfe179e34
Step 5/7 : VOLUME /data
 ---> Running in f93de5446caa
Removing intermediate container f93de5446caa
 ---> 9c174e36bf3a
Step 6/7 : WORKDIR /data
 ---> Running in eed67d591239
Removing intermediate container eed67d591239
 ---> 36cc09b2c59b
Step 7/7 : CMD fortune | cowsay | lolcat
 ---> Running in 87a464d2ee67
Removing intermediate container 87a464d2ee67
 ---> 3c62a0f2e06e
Successfully built 3c62a0f2e06e
Successfully tagged lolcow:1Nov19
```
{: .output}


### A Dockerfile recipe

The image we are building is very similar to the one we built with Singularity.  Let's have a look at its `Dockerfile` recipe file in the demo directory:

```docker
FROM ubuntu:18.04

LABEL maintainer="Pawsey Supercomputing Centre"

RUN apt-get -y update && \
  apt-get -y install fortune cowsay lolcat

ENV PATH=/usr/games:$PATH

VOLUME /data
WORKDIR /data

CMD fortune | cowsay | lolcat
```
{: .source}

The directory where the the Dockerfile is stored is the so called the Docker **build context**.  Docker will include files in this directory in the build process and in the final image.  As a by-product, this will make the build process longer and the image larger, so that we want to include only those strictly required for the build.

Let's comment on the Docker instructions that appear in this Dockerfile.

* `FROM`: compulsory, it provides the starting image we will use to build our customised one;
* `LABEL`: used to add metadata information to the image, *e.g.* the maintainer, optional;
* `RUN`: this is the most used instruction, that allows to run most shell commands during the build.  Multiple `RUN` instructions are often found in a single Dockerfile;
* `ENV`: set environment variables that will persist at runtime in the container; **DO NOT** use `RUN export <..>` to this end, as the variable will be lost after the `RUN` step is completed;
* `VOLUME`: creates a mount point ready to be used for mounting external (*e.g.* host) volumes; creates the corresponding directory if not existing;
* `WORKDIR`: changes directory to the specified path; the last current directory in the build will be the working directory in the running container.  
  **Note**: if you use instead `RUN cd <..>`, the changed directory will only persist within that `RUN` instruction, and then be lost in subsequent build steps;
* `CMD`: specifies the default command to be executed with the container, in case no other command is provided.

More information on the Dockerfile syntax can be found at the [Dockerfile reference](https://docs.docker.com/engine/reference/builder/).


### Layers in a Docker image

Note how the `RUN` instruction above is used to execute a sequence of commands to update the list of available packages and install a set of Linux packages.  
We have concatenated all these commands in one using the `&&` linux operator, and then the `\` symbol to break them into multiple lines for readability.

We could have used one `RUN` instruction per command, so why concatenate instead?  
Well, each `RUN` creates a distinct **layer** in the final image, increasing its size.  It is a good practice to use as few layers, and thus `RUN` instructions, as possible, to keep the image size smaller.


### List local Docker images

We know that Docker container images are not single files, but rather adopt a multi-layered format.  To keep things tidy, Docker stores images and their layers in a hidden directory, under its own control.  
To get the list of available images, including the ones you built, use `docker images`:

```bash
$ sudo docker images
```
{: .source}

```
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
lolcow              1Nov19              a67808f049be        5 hours ago         170MB
```
{: .output}


### What if you need to debug the build process?

Quite often devising a recipe to install software involves a certain deal of trial and error.  

If you need to inspect a Docker container during build, you can open an interactive shell session this way:

```bash
$ sudo docker run --rm -it ubuntu:18.04 bash
```
{: .source}

```
root@dd1ca993f4ad:/#
```
{: .output}

Here, `-it` keeps the container standard input open and allocates a terminal; `--rm` does some clean up when closing the session.  

Note that Docker containers, unlike Singularity containers, are writable.  So during an interactive sessions you can even trial software installations.  However, edits are ephemeral, *i.e.* you lose them when you close the container.

When you're done, type `exit`, or hit `Ctrl-D`, to leave the interactive shell.


### Pushing the image to Docker Hub

If you have a (free) Docker Hub account you must first login to Docker.

```bash
$ sudo docker login
```
{: .source}

You are now ready to push your newly created image to the Docker Hub web registry.

First, let us create a second tag for the image, that includes your Docker Account.  To this end we'll use `docker tag`:

```bash
$ sudo docker tag lolcow:1Nov19 <your-dockerhub-account>/lolcow:1Nov19
```
{: .source}

Now we can push the image:

```bash
$ sudo docker push <your-dockerhub-account>/lolcow:1Nov19
```
{: .source}

```
The push refers to repository [docker.io/marcodelapierre/lolcow]
9d2959e72647: Pushed
317d47a452af: Pushed
e0b3afb09dc3: Mounted from library/ubuntu
6c01b5a53aac: Mounted from library/ubuntu
2c6ac8e5063e: Mounted from library/ubuntu
cc967c529ced: Mounted from library/ubuntu
1Nov19: digest: sha256:295c5695e2b05f6123bc2d8669ec7b66e45df5000ab9fc45ce3566ae3c0d839e size: 1571
```
{: .output}

Your image is now publicly available for anyone to pull.


### Sharing the Docker image as a single file

If you don't want to use an online registry to share your images, Docker allows you to convert them to a compressed `tar.gz` archive, which you can then share as any other large file, *e.g.* using `scp`, `rsync`, and other file transfer tools.  
For instance, this can be useful when needing to transfer or share images including proprietary software, amongst collaborators that own the appropriate license.

Use `docker save` to create the archive:

```bash
$ sudo docker save -o lolcow_1Nov19.tar.gz lolcow:1Nov19
```
{: .source}

After the transfer, use `docker load` to extract the image in a format that is usable by Docker:

```bash
$ sudo docker load -i lolcow_1Nov19.tar.gz
```
{: .source}

```
Loaded image: lolcow:1Nov19
```
{: .output}

**Note**: you need Docker to extract the image from the compressed archive, Singularity can't do it.


### Running the image with Docker

Let's give this image a go! Let's execute it without any argument to use the default command:

```bash
$ sudo docker run --rm lolcow:1Nov19
```
{: .source}

```
 _______________________________________
/ Good news. Ten weeks from Friday will \
\ be a pretty good day.                 /
 ---------------------------------------
        \   ^__^
         \  (oo)\_______
            (__)\       )\/\
                ||----w |
                ||     ||
```
{: .output}

Note how the default command can be readily overwritten:

```bash
$ sudo docker run --rm lolcow:1Nov19 echo "Hello World!"
```
{: .source}

```
Hello World!
```
{: .output}


### Converting and running the image with Singularity

We've created a container image using Docker.  Now, what steps are required to use it with Singularity?  
As seen in a previous episode, the Singularity `pull` command can take care of converting a Docker image for its own usage.

If you have pushed the image to Docker Hub, just execute:

```bash
$ singularity pull docker://<your-dockerhub-account>/lolcow:1Nov19
```
{: .source}

If your Docker-equipped machine also comes with Singularity, you can also grab the image from the local image repo, using the `docker-daemon:` prefix.  Due to a current bug in Singularity, you will need to ditch the double slashes `//`:

```bash
$ singularity pull docker-daemon:lolcow:1Nov19
```
{: .source}

Does it work?

```bash
$ ./lolcow_1Nov19.sif
```
{: .source}

```
 ________________________________________
/ Don't go surfing in South Dakota for a \
\ while.                                 /
 ----------------------------------------
        \   ^__^
         \  (oo)\_______
            (__)\       )\/\
                ||----w |
                ||     ||
```
{: .output}

Sure it does!

### Syntax difference

Most directives in Docker map directly to a Singularity one.

> ## Syntax of recipe files: Docker *vs* Singularity
>
> | Task            |  Docker              |  Singularity         |
> | :-------------- |  :-----------------: |  :-----------------: |
> |                 |  *Directive*         |  *Section*           |
> | :-------------- | :-----------------:  | :-----------------:  |
> | Starting image  |  `FROM`              | `Bootstrap` + `From` |
> | Linux commands  |  `RUN`               | `%post`              |
> | Shell variables |  `ENV`               | `%environment`       |
> | Copying files   |  `COPY`, `ADD`       | `%files`             |
> | Metadata        |  `LABEL`             | `%labels`, `%help`   |
> | Default command |  `CMD, ENTRYPOINT`   | `%runscript`         |
> | Long running    |  Not required        | `%startscript`       |
> | Work directory  |  `WORKDIR`           | Not required         |
> | Mount points    |  `VOLUME`            | Not required         |
{: .callout}


### Bonus: example Dockerfiles

Have a look at these, just to get a taste of what a production Dockerfile might look like.


> ## Pawsey MPI-base image
>
> > ## Dockerfile
> >
> > ```docker
> > # Make sure to add comments all the time
> > # Use ubuntu 18.04, perhaps doesn't work with later versions
> > FROM ubuntu:18.04
> >
> > # Add several labels
> > LABEL maintainer="someone.somewhere@pawsey.org.au"
> > LABEL version="1.0.0"
> > LABEL description="This text illustrates \
> > that label-values can span multiple lines."
> > LABEL this.show-how-complex.a.key.can.be="foo"
> > # Typically containers do not provide extensive metadata but this is poor
> > # practice. Try adding labels that could be useful such as tags
> > LABEL tags="ubuntu/18.04, mpich/3.1.4, osu/5.4.2"
> >
> > # Install package dependencies
> > # note here the inclusion of apt-get clean all and rm -rf
> > # to reduce the size of the image
> > RUN apt-get update -qq \
> >       && apt-get -y --no-install-recommends install \
> >          build-essential \
> >          gdb \
> >          gfortran \
> >          wget \
> >       && apt-get clean all \
> >       && rm -r /var/lib/apt/lists/*
> >
> >
> > ### Build MPICH ###
> >
> > ARG MPICH_VERSION="3.1.4"
> > ARG MPICH_CONFIGURE_OPTIONS="--enable-fast=all,O3 --prefix=/usr"
> > ARG MPICH_MAKE_OPTIONS="-j4"
> >
> > WORKDIR /tmp/mpich-build
> >
> > RUN wget http://www.mpich.org/static/downloads/${MPICH_VERSION}/mpich-${MPICH_VERSION}.tar.gz \
> >       && tar xvzf mpich-${MPICH_VERSION}.tar.gz \
> >       && cd mpich-${MPICH_VERSION}  \
> >       && ./configure ${MPICH_CONFIGURE_OPTIONS} \
> >       && make ${MPICH_MAKE_OPTIONS} \
> >       && make install \
> >       && ldconfig
> >
> >
> > ### Build OSU Benchmarks ###
> >
> > ARG OSU_BENCH_VERSION="5.4.2"
> > ARG OSU_BENCH_CONFIGURE_OPTIONS="--prefix=/usr/local CC=mpicc CXX=mpicxx CFLAGS=-O3"
> > ARG OSU_BENCH_MAKE_OPTIONS="-j4"
> >
> > WORKDIR /tmp/osu-benchmark-build
> >
> > RUN wget http://mvapich.cse.ohio-state.edu/download/mvapich/osu-micro-benchmarks-${OSU_BENCH_VERSION}.tar.gz \
> >       && tar xzvf osu-micro-benchmarks-${OSU_BENCH_VERSION}.tar.gz \
> >       && cd osu-micro-benchmarks-${OSU_BENCH_VERSION} \
> >       && ./configure ${OSU_BENCH_CONFIGURE_OPTIONS} \
> >       && make ${OSU_BENCH_MAKE_OPTIONS} \
> >       && make install
> >
> > WORKDIR /
> > RUN rm -rf /tmp/*
> > CMD ["/bin/bash"]
> > ```
> > {: .source}
> {: .solution}
{: .challenge}


> ## A simple Python image
>
> This docker file lacks useful metadata. Consider adding it.
> > ## Dockerfile
> >
> > ```docker
> > FROM continuumio/miniconda3:4.5.11
> >
> > LABEL maintainer="someone.somewhere@pawsey.org.au"
> >
> > RUN apt-get clean all && \
> >     apt-get update && \
> >     apt-get upgrade -y && \
> >     apt-get install -y \
> >         vim && \
> >     conda update -y conda \
> >     && apt-get clean all && \
> >     apt-get purge && \
> >     rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*
> >
> > ARG atlas_version="2.0.1"
> > RUN conda install -y -c bioconda -c conda-forge metagenome-atlas="$atlas_version"
> >
> > RUN mkdir /databases && \
> >     chmod go+w /databases
> >
> > RUN mkdir /home/none && \
> >     mkdir /home/none/.cache && \
> >     cp -p $HOME/.bashrc $HOME/.profile /home/none/ && \
> >     chmod -R go+w /home/none
> > ENV HOME="/home/none"
> > VOLUME /data
> > WORKDIR /data
> > ```
> > {: .source}
> {: .solution}
{: .challenge}


> ## A large R image
>
> > ## Dockerfile
> >
> > ```docker
> > LABEL maintainer="someone.somewhere@pawsey.org.au"
> > FROM rocker/tidyverse:latest
> >
> > RUN apt-get update -qq && apt-get -y --no-install-recommends install \
> >       autoconf \
> >       automake \
> >       g++ \
> >       gcc \
> >       gfortran \
> >       make \
> >       && apt-get clean all \
> >       && rm -rf /var/lib/apt/lists/*
> >
> > RUN mkdir -p $HOME/.R
> > COPY Makevars /root/.R/Makevars
> >
> > RUN Rscript -e "library('devtools')" \
> >       -e "install_github('Rdatatable/data.table', build_vignettes=FALSE)" \
> >       -e "install.packages('reshape2')" \
> >       -e "install.packages('fields')" \
> >       -e "install.packages('ggbeeswarm')" \
> >       -e "install.packages('gridExtra')" \
> >       -e "install.packages('dynamicTreeCut')" \
> >       -e "install.packages('DEoptimR')" \
> >       -e "install.packages('http://cran.r-project.org/src/contrib/Archive/robustbase/robustbase_0.90-2.tar.gz', repos=NULL, type='source')" \
> >       -e "install.packages('dendextend')" \
> >       -e "install.packages('RColorBrewer')" \
> >       -e "install.packages('locfit')" \
> >       -e "install.packages('KernSmooth')" \
> >       -e "install.packages('BiocManager')" \
> >       -e "source('http://bioconductor.org/biocLite.R')" \
> >       -e "biocLite('Biobase')" \
> >       -e "biocLite('BioGenerics')" \
> >       -e "biocLite('BiocParallel')" \
> >       -e "biocLite('SingleCellExperiment')" \
> >       -e "biocLite('GenomeInfoDb')" \
> >       -e "biocLite('GenomeInfgoDbData')" \
> >       -e "biocLite('DESeq')" \
> >       -e "biocLite('DESeq2')" \
> >       -e "BiocManager::install(c('scater', 'scran'))" \
> >       -e "library('devtools')" \
> >       -e "install_github('IMB-Computational-Genomics-Lab/ascend', ref = 'devel')" \
> >       && rm -rf /tmp/downloaded_packages
> > ```
> > {: .source}
> {: .solution}
{: .challenge}
