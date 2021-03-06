# Build Your Own Pytorch Deep Learning Docker Container for NVidia's Volta GPUs on Amazon EC2 P3 instances 


# 1. Introduction

Nvidia Tesla [V100][1] is the most advanced GPU in the world, and it's now available on-demand on Amazon EC2 [P3 instances][2]. AWS EC2 P3 instances provide a powerful platform for deep learning (DL) and high performance computing (HPC) workloads, with support for up to 8 NVIDIA Tesla V100 GPUs, each containing 5,120 CUDA Cores and 640 Tensor Cores; with 300 GB/s NVLink hyper-mesh interconnect allows GPU-to-GPU communication at high speed and low latency. 

With [`dl-docker-volta`][3] , you get a model for building your own deep learning docker-based workbenches on Volta. Docker is ideal if you want to isolate working environments, yet leverage the same dataset across frameworks. It’s early days in deep learning, and there are many frameworks making it easy to build and train neural networks. All of them improving and adding features, constantly. To take advantage of the entire ecosystem and solve your problems faster, you would want an environment that allows you to scale on-demand for the workloads you’re familiar with, while experimenting faster on those you’re not.

# 2. The Approach

At the root, the project has a folder for each deep learning framework. Each framework has a ```Dockerfile```, an ```entrypoint.sh```, and a ```README.md``` file to get you building Docker images fast on the host machine. The ```Dockerfile``` contains all the required code to install and configure a specific deep learning framework on the host machine. The ```entrypoint.sh``` file contains bootstrap actions to be executed every time a container is launched from an image. In this case, checking availability of GPUs. The ```README.md``` file gives you specifics about building the framework. 

After building the images, you'd want to maintain them on a Docker registry. [Amazon ECR][4] allows you to securely store and share your Docker images on a scalable, managed docker images registry.

Here is the folder structure:

```bash
├── LICENSE
├── README.md
├── mxnet-dlimg
│   └── README.md
├── prepare_ubuntu-xenial-amd64.sh
├── pytorch-dlimg
│   ├── Dockerfile
│   ├── README.md
│   └── entrypoint.sh
└── tensorflow-dlimg
    └── README.md
```

# 3. Preparing the host

To get started, you need to prepare your host machine to run the docker containers. I this case, I used [Ubuntu 16.04 Xenial][5] as the base EC2 image. On it, I installed the following:

1. A Docker local repository
2. Nvidia GPU drivers and plugins
3. ```nvidia-docker``` is a wrapper around docker, which checks for GPU availability in the container and automatically fixes character devices corresponding to Nvidia GPUs (```e.g /dev/nvidia0```).
4. AWS CLI
5. Some helper cli tools I find very useful. [tmux][6] is great for maintaining your sessions in a remote machine intact. If you're like me and you always need a reminder of where some files and libraries are,  ``mlocate`` will be very useful. ``htop`` is great for visualizing the resources utilization of your Linux machine.

The following machine preparation code is available in `prepare_ubuntu-xenial-amd64.sh`, and should be executed as a [user data script][7] on an Amazon EC2 P3 instance.


```bash
#!/bin/bash
#
# Notes:
# This script prepares an Ubuntu16.04 (xenial) machine for building
# deep learning docker containers.
# Run as user-data on EC2.
# User data scripts are executed as root user
# so no need to use sudo.

# Vars
__idVer__="17.10:v1.0"
__author__="Dan R. Mbanga"
VOLTA_BUILD_HOME_DIR="/opt/voltaBuild"

# Mbanga, 17.10

set -e

# Add Nvidia repositories for cuda and machine learning
echo "deb http://developer.download.nvidia.com/compute/machine-learning/repos/ubuntu1604/x86_64 /" > /etc/apt/sources.list.d/nvidia-ml.list
echo "deb http://developer.download.nvidia.com/compute/cuda/repos/ubuntu1604/x86_64 /" > /etc/apt/sources.list.d/cuda.list
curl -fsSL http://developer.download.nvidia.com/compute/cuda/repos/ubuntu1604/x86_64/7fa2af80.pub |  apt-key add -


# Add Docker repository and install Docker community engine

add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -

apt-get update

apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    software-properties-common \
    docker-ce


# Other tools

apt-get install -y git \
    tmux \
    htop \
    apt-utils \
    mlocate \
    python3-pip

pip3 install --upgrade pip
pip3 install --upgrade awscli

# Check your keys with ```apt-key --list```

# Install Nvidia-docker

wget -P /tmp https://github.com/NVIDIA/nvidia-docker/releases/download/v1.0.1/nvidia-docker_1.0.1-1_amd64.deb

dpkg -i /tmp/nvidia-docker_1.0.1-1_amd64.deb

# service nvidia-docker start   # not necessary, reboot at the end.

# Install nvidia-modprobe and nvidia-384 (plugins and drivers)

apt-get install -y nvidia-modprobe nvidia-384

#################### Health check commands ##################
#                                                           #
# journalctl -n -u nvidia-docker                            #
# nvidia-modprobe -u -c=0                                   #
#                                                           #
# Check GPU status on the default nvidia/cuda docker image. #
# nvidia-docker run --rm nvidia/cuda nvidia-smi             #
#                                                           #
#############################################################

# Give user ubuntu access to nvidia-docker and docker binaries
usermod -a -G nvidia-docker ubuntu
usermod -a -G docker ubuntu

############ VOLTA BUILD HOME DIR ######

chmod -R a+w /opt
mkdir -p ${VOLTA_BUILD_HOME_DIR}
chown -R ubuntu:ubuntu ${VOLTA_BUILD_HOME_DIR}

# Update the file locations index
updatedb

## Todo (HOMEWORK FOR YOU): Add the build files (Dockerfile and entrypoint.sh)
##  for each framework you'd like to build, in the ${VOLTA_BUILD_HOME_DIR}

# REBOOT for nvidia plugins and service to take effect

reboot /r

```

Now you have a working Ubuntu 16.04 machine with Nvidia drivers in it. Next, you have to pull the appropriate folder for the framework you'd want to install on a local container. There is a ```Dockerfile``` in each folder that contains the specifications to ```build``` the image on your host machine. In this case, we'll install PyTorch.

Verify that the Nvidia drivers and plugins are installed, and that the service was started succesfully by ```systemd```.

```bash

service nvidia-docker status
● nvidia-docker.service - NVIDIA Docker plugin
   Loaded: loaded (/lib/systemd/system/nvidia-docker.service; enabled; vendor preset: enabled)
   Active: active (running) since Mon 2017-10-30 09:26:32 UTC; 3 days ago
     Docs: https://github.com/NVIDIA/nvidia-docker/wiki
  Process: 1341 ExecStartPost=/bin/sh -c /bin/echo unix://$SOCK_DIR/nvidia-docker.sock > $SPEC_FILE (code=exited, status=0/SUCCESS)
  Process: 1326 ExecStartPost=/bin/sh -c /bin/mkdir -p $( dirname $SPEC_FILE ) (code=exited, status=0/SUCCESS)
 Main PID: 1325 (nvidia-docker-p)
    Tasks: 14
   Memory: 33.8M
      CPU: 6.801s
   CGroup: /system.slice/nvidia-docker.service
           └─1325 /usr/bin/nvidia-docker-plugin -s /var/lib/nvidia-docker

```



Pull the base Nvidia Docker image (```nvidia/cuda:9.0-devel-ubuntu16.04```) which comes with Cuda SDK version 9. This base image will be used by Docker to build the rest of the deep learning machine with PyTorch in it. 

```bash

nvidia-docker run --rm nvidia/cuda:9.0-devel-ubuntu16.04 nvidia-smi
Fri Nov  3 05:54:15 2017
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 384.90                 Driver Version: 384.90                    |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  Tesla V100-SXM2...  Off  | 00000000:00:1E.0 Off |                    0 |
| N/A   41C    P0    23W / 300W |     10MiB / 16152MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID   Type   Process name                             Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+

```

In this case I run the container with a ```--rm``` option and a command ```nvidia-smi```; which downloads the image in my local repository, starts a container and runs the ```nvidia-smi``` commands, then exits and removes the container. To look at your images in the local repository, run ```nvidia-docer images -a```. Remember to always use ```nvidia-docker```. It has all the normal ```docker``` functionalities, with the addition of fixing your GPUs for you, nicely!


# 4. Install PyTorch with examples

## 4.1 Prepare the Build environment

Download the ```pytorch-dlimg``` folder under the ```/opt/voltaBuild``` folder in the host EC2 machine. On the host machine, ```cd /opt/voltaBuild/pytorch-dlimg``` and explore teh ```Dockerfile``` and ```entrypoint.sh``` files that will be required to build the ```PyTorch``` image from the base Nvidia CUDA9 image.

Let's review the Dockerfile:


```docker
FROM nvidia/cuda:9.0-devel-ubuntu16.04
LABEL maintainer="Dan R. Mbanga"
# vars
    ## Python version
ENV PYTHON_VERSION 3.5
ENV CUDNN_VERSION 7.0.3.11
LABEL com.nvidia.cudnn.version="${CUDNN_VERSION}"
    ## The version of this Docker container
ENV DANULAB_PYTORCH_IMAGE_VERSION 17.10
ENV DANULAB_PYTORCH_BUILD_VERSION 1.0
    ## Dan's Anaconda channel for magma-cuda90. I had to build the package to support cuda9.0.
    ## Not yet available on the default Anaconda repository.
ENV DANULAB_ANACONDA_CHANNEL etcshadow

RUN echo 'export PS1="\[\033[01;32m\]\u@\h \[\033[00m\]: \[\033[01;34m\]\w \[\033[00m\]\$"' >> /etc/profile

ENV PYTORCH_WORK_DIR /pytorch-workspace

###############################
# First, install apt-utils

RUN apt-get update

RUN apt-get install -y apt-utils

RUN echo "deb http://developer.download.nvidia.com/compute/machine-learning/repos/ubuntu1604/x86_64 /" > /etc/apt/sources.list.d/nvidia-ml.list

RUN apt-get update

RUN apt-get update && apt-get install -y --no-install-recommends \
            libcudnn7=$CUDNN_VERSION-1+cuda9.0 \
            libcudnn7-dev=$CUDNN_VERSION-1+cuda9.0 \
            build-essential \
            cmake \
            git \
            curl \
            vim \
            tmux \
            mlocate \
            htop \
            ca-certificates \
            libnccl2=2.0.5-3+cuda9.0 \
            libnccl-dev=2.0.5-3+cuda9.0 \
            libjpeg-dev \
            libpng-dev &&\
    rm -rf /var/lib/apt/lists/*

# Install Anaconda full. Not miniconda.
# https://repo.continuum.io/archive/Anaconda3-5.0.1-Linux-x86_64.sh

RUN curl -o ~/anaconda3-latest.sh -O  https://repo.continuum.io/archive/Anaconda3-5.0.1-Linux-x86_64.sh  && \
     chmod +x ~/anaconda3-latest.sh && \
     ~/anaconda3-latest.sh -b -p /opt/conda && \
     rm ~/anaconda3-latest.sh && \
     /opt/conda/bin/conda install conda-build && \
     /opt/conda/bin/conda create -y --name pytorch-py35 python=${PYTHON_VERSION} \
        anaconda numpy pyyaml scipy ipython mkl jupyter && \
     /opt/conda/bin/conda clean -ya

ENV PATH /opt/conda/envs/pytorch-py35/bin:$PATH

##### Installing magma-cuda90 from my Anaconda channel ##################################
# https://anaconda.org/etcshadow/magma-cuda90                                           #
#   To build from source (it takes a while!).                                           #
# WORKDIR /workspace                                                                    #
# RUN git clone https://github.com/pytorch/builder.git                                  #
# RUN cp -r builder/conda/magma-cuda90-2.2.0/ magma-cuda90                              #
# WORKDIR /workspace/magma-cuda90                                                       #
# RUN conda-build .                                                                     #
### Go to /opt/conda/conda-bld/magma-cuda90_1507361009645/  (your version might differ) #
# RUN conda install ./magma-cuda90-2.1.0-h865527f_5.tar.bz2                             #
#########################################################################################


RUN conda install --name pytorch-py35 -c ${DANULAB_ANACONDA_CHANNEL} magma-cuda90

WORKDIR /opt

RUN git clone --recursive https://github.com/pytorch/pytorch.git

WORKDIR /opt/pytorch

RUN git submodule update --init

#################################### BUILDING Pytorch for VOLTA on cuda9.0 ##################
# REF: https://github.com/torch/cutorch/blob/master/lib/THC/cmake/select_compute_arch.cmake #
# REF: https://en.wikipedia.org/wiki/CUDA                                                   #
# VOLTA has compute capabilities 7.0 and 7.1                                                #
# CUDA SDK 9.0 supports compute capability 3.0 through 7.x (Kepler, Maxwell, Pascal, Volta) #
#                                                                                           #
# Here we compile for:                                                                      #
#    - Kepler: 3.7 (AWS P2) +PTX, for the P2                                                #
#    - Maxwell: 5.0 5.2                                                                     #
#    - Jetson TX1: 5.3                                                                      #
#    - Pascal P100: 6.0                                                                     #
#    - Pascal GTX family: 6.1                                                               #
#    - Jetson TX2: 6.2                                                                      #
#    - Volta V100: 7.0+PTX (PTX = parallel Thread Execution): even faster!                  #
#############################################################################################

RUN TORCH_CUDA_ARCH_LIST="3.7+PTX 5.0 5.2 5.3 6.0 6.1+PTX 6.2 7.0+PTX" TORCH_NVCC_FLAGS="-Xfatbin -compress-all" \
    CMAKE_PREFIX_PATH="$(dirname $(which conda))/../" \
    pip install -v .

# Setup the entrypoint.
WORKDIR /opt/pytorch-init

COPY ./entrypoint.sh /opt/pytorch-init/

RUN chmod +x /opt/pytorch-init/entrypoint.sh

# Working directory + examples

WORKDIR ${PYTORCH_WORK_DIR}

RUN git clone --recursive https://github.com/pytorch/examples.git

RUN git clone --recursive https://github.com/pytorch/vision.git && \
    cd vision && pip install -v .

RUN git clone --recursive https://github.com/pytorch/text.git && \
    cd text && pip install -v .

RUN git clone --recursive https://github.com/pytorch/tutorials.git

RUN chmod -R a+w ${PYTORCH_WORK_DIR}

# A reference for exposing jupyter notebooks
EXPOSE 8888

# This helps you easily locate files on the box with ```mlocate <filename>```

RUN updatedb

ENTRYPOINT ["/bin/bash"]

CMD ["/opt/pytorch-init/entrypoint.sh"]
```

A few things happen here:

1. We use the ```nvidia-cuda:9.0-devel-ubuntu16.04``` base image to build our docker environment. This machine comes with CUDA SDK 9.0. The CUDA SDK 9.0 requires CUDNN version 7+ to operate properly. So we install ```libcudnn7``` and ```libcudnn7-dev``` with the appropriate ```${CUDNN_VERSION}```. 


2. CUDA 9.0 also requires ```magma-cuda90``` to accelerate linear algebra operations on Volta. magma-cuda90 isn't yet available on Anaconda default repositories, so we built a version for Volta which is available on my channel [```etcshadow```][8]. The steps to build ```magma-cuda90``` from source are available as comments in the ```Dockerfile```.


3. We install the full version of ```anaconda3``` (you could also consider ```miniconda```, I like having all packages available by default); then create a conda environment for ```PyTorch``` called ```pytorch-python35```, with full ```anaconda3``` loaded, ```python3.5```, extra libraries for data science, and ```mkl``` for math kernel acceleration on CPU.

4. We download and build ```pytorch``` for the following Nvidia GPUs:

- Kepler (AWS P2 instances)
- Maxwell
- Jetson TX1 and TX2
- Pascal P100
- Volta V100

5. Finally, we create a working directory under ```/pytorch-workspace``` where we put pytorch examples.


## 4.2 Build the Docker image

Move to the ```/opt/voltaBuild/pytorch-dlimg/``` folder, and run ```nvidia-docker build -t <image-name>:<tag-name> .``` to build the image. 

In this case, I ran: 

```bash

nvidia-docker build -t danulab/pytorch-dlimg:17.10 -t danulab/pytorch-
dlimg:v1.0 -t danulab/pytorch-dlimg:latest --rm=true .

```

You may want to name your images differently, in my case, I named it ```pytorch-dlimg``` with tags ```17.10``` and ```latest```. You may also want to go get dinner or breakfast because the image build takes about 1.5 hours to complete on a ```p2.2xlarge```.
Once the build is completed succesfully, you will get confirmation messages. I've pasted here the last 3 lines of my build.

```bash
Successfully tagged danulab/pytorch-dlimg:17.10
Successfully tagged danulab/pytorch-dlimg:v1.0
Successfully tagged danulab/pytorch-dlimg:latest
```

## 4.3 Run the Docker Container

Congrats! You now have a running docker environment, built for Volta, with Cuda9 and a pytorch image loaded with Anaconda3, and lots of examples. To view your images, run ```nvidia-docker images -a```. To run an interactive session of your new docker container, do ```nvidia-docker run -it <image-name>```. In my case:

```bash

# Run the docker container in interactive mode.

nvidia-docker run -it  danulab-pytorch 

root@a780acd1a36d:/python-workspace# 

# Activate pytorch-python35 environment
root@a780acd1a36d:/pytorch-workspace# source activate /opt/conda/envs/pytorch-py35/
(/opt/conda/envs/pytorch-py35/) root@a780acd1a36d:/pytorch-workspace# 

# Moving to the examples folder
root@a780acd1a36d:/pytorch-workspace/examples/mnist# cd examples/mnist/
root@a780acd1a36d:/pytorch-workspace/examples/mnist#

```

# 5. Deep Learning at last!

Okay, we did all of this to enjoy the speed of Nvidia Volta on Amazon EC2 P3 instances. It's time to test it!. Move to the examples folder under your workspace in ```/pytorch-workspace/exampels/```, and pick the one you want to execute. I suggest you time it to see how fast it is. Here is how I ran the traditional ```mnist``` example in ```1 min 15 seconds```, or an average processing speed of ```60K images every 7.5 seconds!```. The long and boring log below is for the ```10 Epochs``` or iterations on the entire ```mnist``` dataset. I keep it here so you get a feel of the reality of training deep neural networks :-). For your projects, you will likely go ```50 Epochs``` or more. That's why you'd want a tool like ```tmux``` to start a session, launch the job, then disconnect and reconnect into the same session, at will! 


```bash

# timing the mnist example -- second run. The first run downloads the data.

(/opt/conda/envs/pytorch-py35/) root@a780acd1a36d:/pytorch-workspace/examples/mnist# time python main.py 

main.py:68: UserWarning: Implicit dimension choice for log_softmax has been deprecated. Change the call to include dim=X as an argument.
  return F.log_softmax(x)
Train Epoch: 1 [0/60000 (0%)]	Loss: 2.390087
Train Epoch: 1 [640/60000 (1%)]	Loss: 2.350225
Train Epoch: 1 [1280/60000 (2%)]	Loss: 2.288934
Train Epoch: 1 [1920/60000 (3%)]	Loss: 2.279396
.....
.....
Train Epoch: 10 [58880/60000 (98%)]	Loss: 0.284887
Train Epoch: 10 [59520/60000 (99%)]	Loss: 0.081290

Test set: Average loss: 0.0545, Accuracy: 9817/10000 (98%)

real	1m14.587s
user	1m35.344s
sys	0m14.004s


```

# 6. Conclusion

This fun project will give you your own environment with a clean separation based on Docker. I want to leave you with a few ideas to take this to the next level:

1. Build the ```mxnet``` and ```tensorflow``` images.
2. Add your examples to a local volume on the host and use docker volumes to map the local example to any container.
3. Expose Jupyter notebook on your container, launch it in your entrypoint.sh and ```nvidia-docker run``` your images with a jupyter port (```8888```) binding between the host and the container.


[1]:	https://www.nvidia.com/en-us/data-center/tesla-v100/
[2]:	https://aws.amazon.com/about-aws/whats-new/2017/10/introducing-amazon-ec2-p3-instances/
[3]:    https://github.com/danromuald/dl-docker-volta
[4]:    https://aws.amazon.com/ecr/
[5]:    https://aws.amazon.com/marketplace/pp/B01JBL2M0O
[6]:    https://github.com/tmux/tmux/wiki   
[7]:    http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html
[8]:    https://anaconda.org/etcshadow/
