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

#### Debian-based OS

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

#### Amazon Linux 2 (Graviton 2)

Docker engine can be installed with the following command:

```
sudo amazon-linux-extras install docker
```

### 2.2 docker-compose

There is currently no official docker-compose binary dedicated to ARM64 and compilation from sources will fail due to the absence of necessary Python libraries available in ARM64 architecture. A workaround to solve this problem is as follows:

*a) Install software packages*

#### Debian-based OS

```
sudo apt install python3-dev
sudo apt-get install libffi-dev libssl-dev
sudo apt-get install -y python3 python3-pip
sudo apt-get install build-essential -y
sudo apt-get install python3-dev -y
```

#### Centos/RHEL-based OS

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

```
wget https://golang.org/dl/go1.17.2.linux-arm64.tar.gz

sudo tar -C /usr/local -xzf go1.17.2.linux-arm64.tar.gz 

go --version

export PATH=$PATH:/usr/local/go/bin
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

## **EDIT BASEIMAGE FILES**
The Fabric baseimage repository contains source code for the base docker images required by the fabric repository. Navigate to the **fabric-baseimage**:

```
cd $HOME/go/src/github.com/hyperledger/fabric-baseimage
```

Check the available tagged branches for the fabric-baseimage codebase by executing:

```
git tag
```
For this exercise switch to branch v0.4.20 by executing:
```
git checkout v0.4.20
```

Within the fabric-baseimage codebase there are files that should be adjusted for successful build.

1. Edit the file **config/baseimage/Dockerfile** by commenting out
```
FROM adoptopenjdk:8u222-b10-jdk-openj9-0.15.1
```
and inserting this line:
```
FROM adoptopenjdk/openjdk8:aarch64-ubuntu-jdk8u222-b10
```

2. Edit the file **config/baseos/Dockerfile** by commenting out
```
FROM debian:buster-20190910-slim
```

and inserting this line:
```
FROM arm64v8/debian:buster
```
3. Edit the file **images/couchdb/Dockerfile** by commenting out
```
FROM debian:stretch-20190910-slim
```
and inserting this line;
```
FROM arm64v8/debian:buster
```
Whilst in the same file replace the following line:
```
libicu57 \
```
with the following line
```
libicu63 \
```
Finally in the same file **DELETE** this line as it causes errors within Debain:buster due to the package not being available
```
libmozsf180 \
```
4. Edit the two docker files in **images/zookeeper/Dockerfile** and **images/kafka/Dockerfile** by commenting out
```
FROM debian:buster-20190910-slim as download
```
and insert this line:
```
FROM arm64v8/debian:buster as download
```
still in the same file comment this line
```
FROM adoptopenjdk:8u222-b10-jdk-openj9-0.15.1
```
and insert this line:
```
FROM adoptopenjdk/openjdk8:aarch64-ubuntu-jdk8u222-b10
```
5. Lastly edit the file **scripts/common/setup.sh** and replace two instances of the following line:
```
ARCH=`uname -m | sed 's|i686|386|' | sed 's|x86_64|amd64|'`
```
with this line:
```
ARCH=`uname -m | sed 's|i686|386|' | sed 's|x86_64|amd64|' | sed 's|aarch64|arm64|'`
```
Still in the same file, replace all instances of **golang-1.6** with **golang-1.11**

## BUILDING BASEIMAGE DOCKER IMAGES ##

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

2. Navigate to **fabric directory** and here execute this command to switch to branch v2.1.0
```
git checkout v2.1.0
```

Within this directory Now execute this command:
```
make docker
```

If successful this process will create the **peer, orderer, tools, ccenv**. These individual components can also be built separately e.g ```make peer peer-docker``` or ```make ccenv```.
Run docker images to list all created image.

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

