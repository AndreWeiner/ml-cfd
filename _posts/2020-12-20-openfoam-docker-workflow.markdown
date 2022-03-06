---
layout: post
title:  "A detailed look at the OpenFOAM docker workflow"
date:   2020-12-20 12:00:00 +0100
categories: OpenFOAM Docker
---

**Update (Mar 5 2022)**: the workflow described in this article has been revised at the end of 2022. Have a look at the [updated workflow](https://develop.openfoam.com/Development/openfoam/-/wikis/precompiled/docker). Nonetheless, if you are new to working with Docker and OpenFOAM, you will still learn the essentials from this article.

---

This article is all about

- what the installOpenFOAM script does,
- what the startOpenFOAM script does, and
- ways to ship your OpenFOAM/solvers/utilities using Docker.

The quickest way to get a running OpenFOAM installation on any Linux distribution (or even Mac and Windows) is probably via a Docker image. In case you have never heard of Docker, and you are wondering why you should bother to use it, Robin Knowles from [CFD Engine](https://www.cfdengine.com/) wrote a fantastic article entitled [The complete guide to Docker & OpenFOAM](https://www.cfdengine.com/blog/how-to-install-openfoam-anywhere-with-docker/), which I really recommend to read before continuing with this post. If you are not in the mood to read another article, here is why you should care about Docker in a nutshell:

- Docker allows creating isolated, standardized software images.
- An image contains the software itself and all its dependencies, which is why you can share and run your app (almost) without limit.
- This flexibility makes your app executable even if you do not know the hardware or operating system (OS) it is going to run on. You can run your app in the cloud, or you can share it quickly with a co-worker who uses a different OS, or you can conserve a solver and it will be still runnable in five years from now and it will still give the exact same results.

If you followed the [installation instructions](https://openfoam.com/download/install-binary-linux.php) provided on the ESI OpenFOAM website, you installed Docker and executed two scripts, namely installOpenFOAM, and startOpenFOAM. Afterward, you were magically presented with a running OpenFOAM installation without having had to worry about any dependencies besides Docker. And there is more: in the isolated container environment, you still have the same username as on the Linux host and use the same credentials to run sudo commands. There is also a fully-fledged version of paraFoam available. You may be wondering why this should be a big deal?! Well, Docker is great for creating standardized, isolated, and minimalistic environments. Isolation means that Docker only uses few core components of the host system’s Linux kernel (Cgroups, Namespaces, etc. - [learn more](https://docs.docker.com/engine/security/security/)), but it doesn’t depend on or interact with any other applications, libraries, or configuration files of the host. To work in a Docker container feels a bit like working on a remote server, with the difference being only that this remote server is an isolated fraction of your workstation. Still, with the OpenFOAM-plus workflow, the OpenFOAM container integrates seamlessly into your system, and you can run simulations just as well as with the native installation. But how and when does the mapping of username, password, or permissions happen? And how does it become possible to access your simulation data if created in an isolated environment? It all happens with the execution of two short scripts. Understanding these scripts will enable you to modify them according to your needs. If you are curious to learn more, read on.

### *installOpenFOAM*

Let’s start with the first script that was executed: `installOpenFOAM`. The name suggests that this script installs OpenFOAM on your computer, but as you will learn in the next paragraphs, there is no classical installation process when working with images/containers. A more suitable name might be `initOpenFOAMContainer` or even better `runOpenFOAMContainer`, but I guess such naming could confuse users new to Docker and containerization. The script may be divided into two logical parts: first, some useful environment variables are defined, and then the Docker run command is executed.

```bash
username="$USER"
user="$(id -u)"
home="${1:-$HOME}"
imageName="openfoamplus/of_v1812_centos73"
containerName="of_v1812"   
displayVar="$DISPLAY"
```

Line 1 and 2 define variables for username and user id. The username will be your login name as in the command line prompt, e. g. **username**`@workstation:~$`. The id is an integer value associated with the user, most likely 1001. The next line is of great importance because it defines where (in which path) you will interact with the OpenFOAM container. The syntax of the right-hand-side works as follows: `${defined_path:-default_path}`. The default path is simply your home directory `HOME`. The default path is used if no other valid path was given as the first command-line argument to the `installOpenFOAM` script. To change the default path, one would execute `./installOpenFOAM /absolute/alternative/path`. The imageName is the name of the OpenFOAM image hosted on [Dockerhub](https://hub.docker.com/r/openfoamplus/of_v1812_centos73). The part of the image name before the front slash corresponds to the Dockerhub user, here **openfoamplus**. The second part gives more information about the image. For example, the present image is built on CentOS 7.3 and contains version 1812 of OpenFOAM-plus. Last but not least, there is the `DISPLAY` variable, which tells an application with a graphical user interface (GUI) where to display the interface. On my laptop, the value of `DISPLAY` is simply `:0` (zero), which is my primary (and only) screen. Remember, Docker containers are a bit like remote servers, so you have to provide some additional information to use GUI applications. With all the information gathered up to this point, we are ready to move on to the actual container creation.

The syntax to create and execute a container is `docker run [options] IMAGE [command] [args]` ([read more](https://docs.docker.com/engine/reference/run/)). The image name is stored in the variable `imageName` and appears only in the twelfth line of the command in the code box below. Every item between docker run and ${imageName} starting with `-` or `--` is an option. The command to be executed after the container was created is /bin/bash, which is why you are presented with a command-line prompt when starting the OpenFOAM container. Bash is run with the `-rcfile` argument to execute additional commands from the file specified thereafter (line 13). The content of `setImage.sh` is shown in the last code box of this article for completeness.

{% highlight bash linenos %}
docker run  -it -d --name ${containerName} --user=${user}   \
    -e USER=${username}                                     \
    -e QT_X11_NO_MITSHM=1                                   \
    -e DISPLAY=${displayVar}                                \
    -e QT_XKB_CONFIG_ROOT=/usr/share/X11/xkb                \
    --workdir="${home}"                                     \
    --volume="${home}:${home}"                              \
    --volume="/etc/group:/etc/group:ro"                     \
    --volume="/etc/passwd:/etc/passwd:ro"                   \
    --volume="/etc/shadow:/etc/shadow:ro"                   \
    --volume="/etc/sudoers.d:/etc/sudoers.d:ro"             \
     -v=/tmp/.X11-unix:/tmp/.X11-unix ${imageName}          \
     /bin/bash --rcfile /opt/OpenFOAM/setImage_v1812.sh
{% endhighlight %}

Let’s take a closer look at all the container options displayed above because this is where the magic happens.

| option | description |
|:--:|:--|
| `-i` or `--interactive` | keeps the standard input open even if you detach from the container, e.g., if you close the terminal |
| `-t` or `--tty` | allocates a virtual console to interact with the container |
| `-d` or `--detach` | allows running a container in the background; the container will not stop after you exit from the container |
| `--name` | sets a container name |
| `-u` or `--user` | creates an additional user within the container; the default user is `root` |
| `-e` or `--env` | sets environment variables in the container |
| `-w` or `--workdir` | sets the working directory inside the container, e.g., the default directory after attaching (logging in) to the container |
| `-v` or `--volume` | binds a source from the host to the container; such a source might be a file, a directory, or a Docker volume |

The first interesting option is the `--user` flag, which is used to create a new user inside the container with the same id as the user creating the container. Additionally, in line 2, the `-e` option sets the corresponding username. More environment variables are set in lines 3-5 to enable a GUI (mainly ParaView) to be forwarded to the host system. In line 6 the working directory is set to `home`. The home directory will be the same as on the host system since we first mapped the user to the container (unless a different directory was specified). The syntax to [bind volumes](https://docs.docker.com/storage/bind-mounts/) to the container is `--volume path_on_host:path_in_container:options`. The last argument is optional, for example, to make a file or directory read-only with the `ro` option. The first and most important directory-mount happens in line 7, where the home directory of the container is bound to the host’s home. If you use the `FOAM_RUN` directory to run test cases, then all the solver/utility output will be accessible from the home directory of the host. Likewise, data can be made accessible to the container by moving it to the home directory. After mounting home, there are three more single files and two folders that are bound to the container:

- the `/etc/group` file contains a list of groups and their members
- `/etc/passwd` contains further user attributes like id or password
- `/etc/shadow` is a file with the same content as `/etc/passwd` but only root has read access (this is some safety feature of modern Linux systems)
- the `sudoers.d` folder sometimes contains sudoers (users with root privileges) information that has to stay unchanged whenever the system is upgraded
- the `.X11-unix` folder contains an endpoint (a Unix socket) for the Xserver to communicate with clients (applications like ParaView)

### *startOpenFOAM*

The `installOpenFOAM` will be only run once to create the OpenFOAM container. `startOpenFOAM` is the script to execute whenever you need to login to the container. The first line in the scripts grants the container access to the Xserver of the host (to draw GUIs). After that follow two common Docker commands. `docker start CONTAINER_NAME` will start the container in case it was stopped, for example, after rebooting the host system. It is important to note, however, that the container must exist. Finally, an interactive bash shell is executed in the running container using the Docker `exec` command: `docker exec [options] CONTAINER_NAME command [args]`.

```bash
xhost +local:of_v1812
docker start of_v1812
docker exec -it of_v1812 /bin/bash -rcfile /opt/OpenFOAM/setImage_v1812.sh
```

The content of the `rcfile` loaded when you execute bash in the container is included in the code box below. First, all the OpenFOAM-specific variables and commands are sourced (made available). After that, some third-party binaries and libraries are added to `PATH` and `LD_LIBRARY_PATH` to have them system-wide available for execution or for compiling new applications (in the container). The last exported variable is again a dependency of `paraFoam`, which is built with [Qt](https://en.wikipedia.org/wiki/Qt_(software)).

```bash
source /opt/OpenFOAM/OpenFOAM-v1812/etc/bashrc
export LD_LIBRARY_PATH=$WM_THIRD_PARTY_DIR/platforms/linux64Gcc/ParaView-5.6.0/lib/mesa:$LD_LIBRARY_PATH
export LD_LIBRARY_PATH=$WM_THIRD_PARTY_DIR/platforms/linux64Gcc/ParaView-5.6.0/lib/paraview-5.6/plugins:$LD_LIBRARY_PATH
export LD_LIBRARY_PATH=$WM_THIRD_PARTY_DIR/platforms/linux64Gcc/qt-5.9.0/lib:$LD_LIBRARY_PATH
export LD_LIBRARY_PATH=$WM_THIRD_PARTY_DIR/platforms/linux64/zlib-1.2.11/lib:$LD_LIBRARY_PATH
export PATH=$WM_THIRD_PARTY_DIR/platforms/linux64Gcc/qt-5.9.0/bin:$PATH
export QT_PLUGIN_PATH=$WM_THIRD_PARTY_DIR/platforms/linux64Gcc/qt-5.9.0/plugins
```

### Shipping OpenFOAM using Docker

If you have your own modified version of OpenFOAM or some solver/utility you have written, here are approaches to preserve and ship your work using Docker:

- Large modifications to OpenFOAM itself require to build a new OpenFOAM Docker image. For that you can base your image on a Linux distribution close to your development environment, e.g. Ubuntu or CentOS, you install all dependencies needed to compile OpenFOAM from scratch, you copy your modified version to the image, and compile it.
- A new solver/utility can be built on top of the official OpenFOAM Docker image. You will copy the new sources to the image, compile them, and voilà.
- If you only need a runnable binary of your app, it is possible to build an image from scratch and to copy only the binary and the dynamically-linked libraries to the image. That’s the most minimalistic way to ship an app.

I will describe each of these approaches in more detail in follow-up articles.

Cheers, Andre