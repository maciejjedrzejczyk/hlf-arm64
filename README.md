# Deployment of Hyperledger Fabric on ARM64 architecture

## 1. DOCUMENT OVERVIEW

This guide will take you through the process of deploying Hyperledger Fabric on devices using ARM64 architecture. The following areas will be covered in the tutorial below:

- *Section 1. Document Overview*
- *Section 2. Installation of prerequisites on your development device*
- *Section 3. Preparation of development environment and downloading the necessary git repositories*
- *Section 4. Preparation of Docker images*
- *Section 5. Compilation of Hyperledger Fabric binaries*
- *Section 6. Deployment of a sample Hyperledger Fabric network using `fabric-samples` repository from Hyperledger.*

### 1.1 Verified deployments

This guide has been tested successfully on Raspberry Pi 4 (DietPi) and AWS Graviton 2 EC2 instance (Amazon Linux 2) with following Hyperledger Fabric components:

- *Hyperledger Fabric 2.3.3 (vanilla)*
- *Fabric-CA 1.5.2 (vanilla)*
- *Fabric-nodeenv 2.3.3 (vanilla)*

### 1.2 References

- *Hyperledger-Fabric-ARM64-images by Chinyati* : https://github.com/chinyati/Hyperledger-Fabric-ARM64-images
- *Hyperledger Fabric Documentation*: https://hyperledger-fabric.readthedocs.io/

### 1.3 To Do

- *Testing the deployment on Apple M1 CPU*
- *Creation of Hyperledger Caliper ARM-compatible images and testing with Hyperledger Fabric*
- *Creation of Hyperledger Explorer ARM-compatible images and testing with Hyperledger Fabric*

## 2 PREREQUISITES

In order to proceed with the deployment of Hyperledger Fabric on ARM64 device and test it against `fabric-samples` scripts, it is necessary to install the following components:

### 2.1 Docker engine

* a) Debian-based OS *

The installation of this component is fairly similar to its counterpart on x86 architecture:

```
sudo apt-get install     apt-transport-https     ca-certificates     curl     gnupg     lsb-release

curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo   "deb [arch=arm64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian \
$(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update

sudo apt-get install docker-ce docker-ce-cli containerd.io -y

sudo groupadd docker

sudo usermod -aG docker $USER

logout
```

* b) Amazon Linux 2 (Graviton 2)*

Docker engine can be installed with the following command:

```
sudo amazon-linux-extras install docker
```

* c) MacOS*

Docker for Mac using M1 processor can be downloaded from an official Docker website: https://docs.docker.com/desktop/mac/install/

This installer will also include `docker-compose`.

### 2.2 docker-compose

There is currently no official docker-compose binary dedicated to ARM64 and compilation from sources will fail due to the absence of necessary Python libraries available in ARM64 architecture. A workaround to solve this problem is as follows:

*a) Install software packages*

*Debian-based OS*

```
sudo apt install python3-dev
sudo apt-get install libffi-dev libssl-dev
sudo apt-get install -y python3 python3-pip
sudo apt-get install build-essential -y
sudo apt-get install python3-dev -y
```

*Centos/RHEL-based OS*

```
sudo yum groupinstall "Development Tools"
sudo yum install python3-devel
```

*b) Compile libsodium library*

```
wget https://download.libsodium.org/libsodium/releases/libsodium-1.0.18-stable.tar.gz

tar -xvf libsodium-1.0.18-stable.tar.gz libsodium-stable

cd libsodium-stable/

./configure 

make -j

make -j check

sudo make install
```

*c) Install pynacl and docker-compose*

```
SODIUM_INSTALL=system pip3 install pynacl

SODIUM_INSTALL=system pip3 install docker-compose

docker-compose -v
```

#### Golang

The installation of this component is fairly similar to its counterpart on x86 architecture. The newest release of Golang can be used.

*a) Debian and Centos/RHEL-based based OS*

```
wget https://golang.org/dl/go1.17.2.linux-arm64.tar.gz

sudo tar -C /usr/local -xzf go1.17.2.linux-arm64.tar.gz 

go --version

export PATH=$PATH:/usr/local/go/bin
```

*b) MacOS*

Golang can be install with Brew:

```
brew install go
```

#### Python 3.x

Usually there is no need to install Python 3.x because it comes preinstalled with Operating Systems which were tested in this guide.

## 3 PREPARING THE DEVELOPMENT ENVIRONMENT

### 3.1 Preparing the development environment and environmental variables

The Hyperledger Fabric Docker images need to be built within a Go workspace which should be created after installing dependencies. To manually create workspace, do the following:

```
mkdir -p $HOME/golang

export GOPATH=$HOME/go
```

Check whether path has been added by executing `echo $PATH`. If the `GOPATH` has not been added open the `~/.profile` or `~/.bashrc` and add the `GOPATH`, then execute `source ~/.bashrc` to save the paths.

Verify Golang environment setup by checking `go env` and see if `GOPATH` and `GOROOT` have been setup.

### 3.2 Clone repositories

To successfully complete the deployment of Hyperledger Fabric, it will be necessary to download the following git repositories:

*Create a single directory for all deployments:*

```
mkdir -p $HOME/go/src/github.com/hyperledger
```

*Go to the folder:*

```
cd $HOME/go/src/github.com/hyperledger
```

*Clone the following repositories*

```
git clone https://github.com/hyperledger/fabric-baseimage.git
git clone https://github.com/hyperledger/fabric.git
git clone https://github.com/hyperledger/fabric-ca.git
git clone https://github.com/hyperledger/fabric-chaincode-node
```

## 4. PREPARATION OF DOCKER IMAGES

### 4.1 baseimage, baseos, kafka, zookeeper and couchdb

The Fabric baseimage repository contains source code for the base docker images required by the fabric repository. Navigate to the **fabric-baseimage**:

```
cd $HOME/go/src/github.com/hyperledger/fabric-baseimage
```

#### 4.1.1 Setting a target branch

Check the available tagged branches for the fabric-baseimage codebase by executing:

```
git tag
```
For this exercise switch to branch v0.4.22 by executing:
```
git checkout v0.4.22
```

Within the fabric-baseimage codebase there are files that should be adjusted for successful build.

#### 4.1.2 Fabric baseimage

Edit the file **config/baseimage/Dockerfile** with your favourite text editor:

```
nano ./config/baseimage/Dockerfile
```

Comment out the following line:

```
FROM adoptopenjdk:8u222-b10-jdk-openj9-0.15.1
```

and replace it with the following line:

```
FROM adoptopenjdk/openjdk8:aarch64-ubuntu-jdk8u302-b08
```

#### 4.1.3 Fabric baseos

Edit the file **config/baseos/Dockerfile**:


```
nano ./config/baseos/Dockerfile 
```

Comment out the following line:

```
FROM debian:buster-20190910-slim
```

and replace it with the following line:

```
FROM arm64v8/debian:buster
```

#### 4.1.4 Couchdb (Hyperledger Fabric-branded Docker image)

Please note that this image is no longer used by Hyperledger Fabric 2.x and instead, a Apache-maintained Couchdb Docker image is used instead. The documentation on how to prepare a Dockerfile for ARM64 of Fabric-maintained Couchdb Docker image is here for posterity only.

Edit the file **images/couchdb/Dockerfile**:

```
nano ./images/couchdb/Dockerfile
```

First, comment out the following line:

```
FROM debian:buster-20190910-slim
```

and replace it with the following line:

```
FROM arm64v8/debian:buster
```

Then, put the following set of installation commands betwen `RUN groupadd` and `RUN apt-get update`:


```
RUN apt-get update -y && apt-get install -y --no-install-recommends \
    wget \
    libnspr4-dev

RUN wget http://ftp.br.debian.org/debian/pool/main/libf/libffi/libffi6_3.2.1-6_arm64.deb
RUN wget http://ftp.ro.debian.org/debian/pool/main/m/mozjs/libmozjs185-1.0_1.8.5-1.0.0+dfsg-6_arm64.deb
RUN wget http://ftp.br.debian.org/debian/pool/main/m/mozjs/libmozjs185-dev_1.8.5-1.0.0+dfsg-6_arm64.deb
RUN wget http://ftp.pl.debian.org/debian/pool/main/libf/libffi/libffi-dev_3.2.1-6_arm64.deb

RUN dpkg -i libffi6_3.2.1-6_arm64.deb
RUN dpkg -i libffi-dev_3.2.1-6_arm64.deb
RUN dpkg -i libmozjs185-1.0_1.8.5-1.0.0+dfsg-6_arm64.deb
```

Comment out the following line:

```
libicu57 \
```

and replace it with the following line:

```
libicu63 \
```

Finally in the same file comment out the following lines:

```
libmozjs185 \
libnspr4-dev \
```


Then, put the following set of installation commands betwen `&& rm -rf /var/lib/apt/lists/*` and `ENV GOSU_VERSION 1.10`:


```
RUN dpkg -i libmozjs185-dev_1.8.5-1.0.0+dfsg-6_arm64.deb
```

Later within this file, comment out the following line:

```
libmozjs185-dev \
```

#### 4.1.5 Kafka and Zookeeper

Please note that Kafka and Zookeeper Ordering Service Network is set to deprecated status in Hyperledger Fabric 2.x and instead, Raft OSN is used instead. The documentation on how to prepare a Dockerfile for ARM64 of Kafka/Zookeeper Docker images is here for posterity only.

Edit the two docker files in **images/zookeeper/Dockerfile** and **images/kafka/Dockerfile**:

for Zookeeper:

```
nano ./images/zookeeper/Dockerfile
```

for Kafka:

```
nano ./images/kafka/Dockerfile
```

First, comment out the following line:

```
FROM debian:buster-20190910-slim as download
```

and insert this line:

```
FROM arm64v8/debian:buster as download
```

still in the same file comment this line:

```
FROM adoptopenjdk:8u222-b10-jdk-openj9-0.15.1
```

and insert this line:

```
FROM adoptopenjdk/openjdk8:aarch64-ubuntu-jdk8u302-b08
```

#### 4.1.6 Makefile scripts

Lastly edit the file **scripts/common/setup.sh**:

```
nano ./images/zookeeper/Dockerfile
```

replace two instances of the following line:
```
ARCH=`uname -m | sed 's|i686|386|' | sed 's|x86_64|amd64|'`
```
with this line:
```
ARCH=`uname -m | sed 's|i686|386|' | sed 's|x86_64|amd64|' | sed 's|aarch64|arm64|'`
```
Still in the same file, comment out the following part of the script:

```
#else
   # Install Golang 1.6 binary as a bootstrap to compile the Golang GO_VER source
#   apt-get -y install golang-1.6
#apt-get -y install golang-1.11
#   cd /tmp
#   wget --quiet --no-check-certificate https://storage.googleapis.com/golang/go${GO_VER}.src.tar.gz
#   tar -xzf go${GO_VER}.src.tar.gz -C /opt

#   cd $GOROOT/src
#   export GOROOT_BOOTSTRAP="/usr/lib/go-1.6"
#    export GOROOT_BOOTSTRAP="/usr/lib/go-1.11"
#   ./make.bash
#   apt-get -y remove golang-1.6
#apt-get -y remove golang-1.11
```

#### 4.1.7 Building Baseimage Docker images

After setting up the Prerequisites and also editing the files in the Base-image codebase its time to make the images. First check if there are any available docker images and if any remove them.

1. The following instructions should be done whilst in **fabric-baseimage directory**

* To build docker images for the baseimage and baseos execute:

```
make docker
```

After an hour+ build should complete and then run **docker images** to list the successfully built images. If successful proceed.

* To build docker image for couchdb execute:
```
make couchdb
```

* To build docker image for kafka execute:
```
make kafka
```

* To build docker image for zookeeper execute:

```
make zookeeper
```

At this point run **docker images** to see created images. If successful a list of created docker images will show.

```
REPOSITORY                     TAG            IMAGE ID       CREATED          SIZE
hyperledger/fabric-zookeeper   arm64-0.4.22   d2da71e0aac2   4 minutes ago    374MB
hyperledger/fabric-zookeeper   latest         d2da71e0aac2   4 minutes ago    374MB
hyperledger/fabric-kafka       arm64-0.4.22   6d3216ecbd04   4 minutes ago    368MB
hyperledger/fabric-kafka       latest         6d3216ecbd04   4 minutes ago    368MB
hyperledger/fabric-couchdb     arm64-0.4.22   8c91b8547d19   5 minutes ago    469MB
hyperledger/fabric-couchdb     latest         8c91b8547d19   5 minutes ago    469MB
hyperledger/fabric-baseimage   arm64-0.4.22   c75c8c828175   35 minutes ago   1.11GB
hyperledger/fabric-baseimage   latest         c75c8c828175   35 minutes ago   1.11GB
hyperledger/fabric-baseos      arm64-0.4.22   ffc7104b983f   48 minutes ago   120MB
hyperledger/fabric-baseos      latest         ffc7104b983f   48 minutes ago   120MB

```

### 4.2 peer, orderer, tools, ccenv, fabric binaries

The Fabric fabric repository contains source code for the docker images required for peer, orderer, ccenv tools and fabric binaries. Navigate to the **fabric**:

```
cd $HOME/go/src/github.com/hyperledger/fabric
```

#### 4.2.1  Setting a target branch

Navigate to **fabric directory** and here execute this command to switch to branch v2.3.0
```
git checkout v2.3.3
```

#### 4.2.2 peer

The original `golang-alpine` docker image used by the `peer` component fails to build with `peer` binary inside due to lack of necessary libraries. Instead, it is necessary to leverage `golang-buster` image for ARM64. The output will result with a slightly bigger but properly functioning `peer` docker container.

Move the original Dockerfile to make a room for a new file:

```
mv images/peer/Dockerfile images/peer/Dockerfile.old
```

Create a new Dockerfile:

```
nano images/peer/Dockerfile
```

Now, include the following text:

```
ARG GO_VER
ARG ALPINE_VER

FROM golang:${GO_VER}-buster as golang

RUN apt-get update \
    && apt-get install -y curl \
    git \
    gcc \
    make \
    musl-dev \
    bash \
    binutils ;

ADD . $GOPATH/src/github.com/hyperledger/fabric
WORKDIR $GOPATH/src/github.com/hyperledger/fabric

FROM golang as peer
ARG GO_TAGS
RUN make peer GO_TAGS=${GO_TAGS}

FROM golang:${GO_VER}-buster
ENV FABRIC_CFG_PATH /etc/hyperledger/fabric
VOLUME /etc/hyperledger/fabric
VOLUME /var/hyperledger
COPY --from=peer /go/src/github.com/hyperledger/fabric/build/bin /usr/local/bin
COPY --from=peer /go/src/github.com/hyperledger/fabric/sampleconfig/msp ${FABRIC_CFG_PATH}/msp
COPY --from=peer /go/src/github.com/hyperledger/fabric/sampleconfig/core.yaml ${FABRIC_CFG_PATH}
EXPOSE 7051
CMD ["peer","node","start"]

```

Save the file and exit the folder.


#### 4.2.3 orderer

There is no need for modification of an original Dockerfile. You can now go to another step.

#### 4.2.4 ccenv

There is no need for modification of an original Dockerfile. You can now go to another step.

#### 4.2.5 tools

Similarly to peer container, the original `golang-alpine` docker image used by `fabric-tools` image fails to build with `peer` binary inside due to lack of necessary libraries for ARM64. Instead, it is necessary to leverage `golang-buster` image. The output will result with a slightly bigger but properly functioning `peer` docker container.

Move the original Dockerfile to make a room for a new file:

```
mv images/tools/Dockerfile images/tools/Dockerfile.old
```

Create a new Dockerfile:

```
nano images/tools/Dockerfile
```


Now, include the following text:

```
ARG GO_VER
#ARG ALPINE_VER
#FROM golang:${GO_VER}-alpine${ALPINE_VER} as golang
FROM golang:${GO_VER}-buster as golang

RUN apt-get update \
    && apt-get install -y curl \
    git \
    gcc \
    make \
    musl-dev \
    bash \
    tar \
    binutils \
    gnupg;

ADD . $GOPATH/src/github.com/hyperledger/fabric
WORKDIR $GOPATH/src/github.com/hyperledger/fabric

FROM golang as tools
ARG GO_TAGS
RUN make peer configtxgen configtxlator cryptogen discover osnadmin idemixgen GO_TAGS=${GO_TAGS}

FROM golang:${GO_VER}-buster

RUN apt-get update \
    && apt-get install -y jq \
    tzdata \
    git \
    bash;

ENV FABRIC_CFG_PATH /etc/hyperledger/fabric
VOLUME /etc/hyperledger/fabric
COPY --from=tools /go/src/github.com/hyperledger/fabric/build/bin /usr/local/bin
COPY --from=tools /go/src/github.com/hyperledger/fabric/sampleconfig ${FABRIC_CFG_PATH}

```

Save the file.

#### 4.2.6 Building peer, orderer, tools and ccenv Docker images and binaries

Within this directory Now execute this command:

```
make docker
```

If successful this process will create the **peer, orderer, tools, ccenv**. These individual components can also be built separately e.g ```make peer peer-docker``` or ```make ccenv```.

At this point run **docker images** to see created images. If successful a list of created docker images will be updated as follows:

```
REPOSITORY                     TAG                              IMAGE ID       CREATED          SIZE
hyperledger/fabric-tools       2.3                              8a606f23287c   2 minutes ago    872MB
hyperledger/fabric-tools       2.3.3                            8a606f23287c   2 minutes ago    872MB
hyperledger/fabric-tools       arm64-2.3.3-snapshot-99553020d   8a606f23287c   2 minutes ago    872MB
hyperledger/fabric-tools       latest                           8a606f23287c   2 minutes ago    872MB
hyperledger/fabric-peer        2.3                              99837ca3072b   3 minutes ago    775MB
hyperledger/fabric-peer        2.3.3                            99837ca3072b   3 minutes ago    775MB
hyperledger/fabric-peer        arm64-2.3.3-snapshot-99553020d   99837ca3072b   3 minutes ago    775MB
hyperledger/fabric-peer        latest                           99837ca3072b   3 minutes ago    775MB
hyperledger/fabric-orderer     2.3                              cc10d618ae77   4 minutes ago    33.3MB
hyperledger/fabric-orderer     2.3.3                            cc10d618ae77   4 minutes ago    33.3MB
hyperledger/fabric-orderer     arm64-2.3.3-snapshot-99553020d   cc10d618ae77   4 minutes ago    33.3MB
hyperledger/fabric-orderer     latest                           cc10d618ae77   4 minutes ago    33.3MB
hyperledger/fabric-ccenv       2.3                              abb88bd95e67   5 minutes ago    496MB
hyperledger/fabric-ccenv       2.3.3                            abb88bd95e67   5 minutes ago    496MB
hyperledger/fabric-ccenv       arm64-2.3.3-snapshot-99553020d   abb88bd95e67   5 minutes ago    496MB
hyperledger/fabric-ccenv       latest                           abb88bd95e67   5 minutes ago    496MB
hyperledger/fabric-baseos      2.3                              49e4c027a401   5 minutes ago    6.65MB
hyperledger/fabric-baseos      2.3.3                            49e4c027a401   5 minutes ago    6.65MB
hyperledger/fabric-baseos      arm64-2.3.3-snapshot-99553020d   49e4c027a401   5 minutes ago    6.65MB
hyperledger/fabric-baseos      latest                           49e4c027a401   5 minutes ago    6.65MB
hyperledger/fabric-zookeeper   arm64-0.4.22                     d2da71e0aac2   14 minutes ago   374MB
hyperledger/fabric-zookeeper   latest                           d2da71e0aac2   14 minutes ago   374MB
hyperledger/fabric-kafka       arm64-0.4.22                     6d3216ecbd04   15 minutes ago   368MB
hyperledger/fabric-kafka       latest                           6d3216ecbd04   15 minutes ago   368MB
hyperledger/fabric-couchdb     arm64-0.4.22                     8c91b8547d19   15 minutes ago   469MB
hyperledger/fabric-couchdb     latest                           8c91b8547d19   15 minutes ago   469MB
hyperledger/fabric-baseimage   arm64-0.4.22                     c75c8c828175   45 minutes ago   1.11GB
hyperledger/fabric-baseimage   latest                           c75c8c828175   45 minutes ago   1.11GB
hyperledger/fabric-baseos      arm64-0.4.22                     ffc7104b983f   58 minutes ago   120MB
```

3. Dependent on what the Raspberry Pi will be in your deployment create the fabric-ca by navigating to **fabric-ca** directory. In this directory switch to branch v1.4.7
```
git checkout v1.4.7
```

Open the Makefile in fabric-ca and replace every instance of amd64 to arm64 for Linux entries.

Within this directory Now execute this command:
```
make docker
```

## BUILDING BASEIMAGE DOCKER IMAGES ##

Binary executables which include peer, orderer, cryptogen and more need to be built for the ARM64 architecture as well. These can be built from the source code in /fabric folder.
```
$ make native
```

These built executables will be places in the bin folder within /bin and must be moved to the /bin folder for the Hyperledger fabric samples or project.

### **ALL REQUIRED DOCKER IMAGES WOULD HAVE BEEN BUILT BY NOW.**
These built images can be accessed from https://hub.docker.com/u/chinyati.
Run docker images to view them.

ERRORS
For errors that could come up when deploying test-network or your channels look for core.yaml file in config and edit Memory value to:
```
Memory: 16777216
```

Install gcc to avoid runtime errors
```
sudo apt install -y gcc
```

