# Deployment of Hyperledger Fabric on ARM64 architecture

## Introduction

This guide will take you through the process of deploying Hyperledger Fabric on devices using ARM64 architecture. The following areas will be covered in the tutorial below:

- installation of prerequisites on your development device
- preparation of development environment and downloading the necessary git repositories
- preparation of Docker images
- compilation of Hyperledger Fabric binaries
- deployment of a sample Hyperledger Fabric network using `fabric-samples` repository from Hyperledger.

## Verified deployments

This guide has been tested successfully on Raspberry Pi 4 (DietPi) and AWS Graviton 2 EC2 instance (Amazon Linux 2) with following Hyperledger Fabric components:

- Hyperledger Fabric 2.3.3 (vanilla)
- Fabric-CA 1.5.2 (vanilla)
- Fabric-nodeenv 2.3.3 (vanilla)

## References

- Hyperledger-Fabric-ARM64-images by Chinyati : https://github.com/chinyati/Hyperledger-Fabric-ARM64-images
- Hyperledger Fabric Documentation: https://hyperledger-fabric.readthedocs.io/

## To Do

- Testing the deployment on Apple M1 CPU
- Creation of Hyperledger Caliper ARM-compatible images and testing with Hyperledger Fabric
- Creation of Hyperledger Explorer ARM-compatible images and testing with Hyperledger Fabric

## Prerequisites

In order to proceed with the deployment of Hyperledger Fabric on ARM64 device and test it against `fabric-samples` scripts, it is necessary to install the following components:

### Docker engine

- Debian-based OS

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

- Amazon Linux 2 (Graviton 2)

Docker engine can be installed with the following command:

```
sudo amazon-linux-extras install docker
```

### docker-compose

There is currently no official docker-compose binary dedicated to ARM64 and compilation from sources will fail due to the absence of necessary Python libraries available in ARM64 architecture. A workaround to solve this problem is as follows:

a) Install software packages

- Debian-based OS

```
sudo apt install python3-dev
sudo apt-get install libffi-dev libssl-dev
sudo apt-get install -y python3 python3-pip
sudo apt-get install build-essential -y
sudo apt-get install python3-dev -y
```

- Centos/RHEL-based OS

```
sudo yum groupinstall "Development Tools"
sudo yum install python3-devel
```

b) Compile libsodium library

```
wget https://download.libsodium.org/libsodium/releases/libsodium-1.0.18-stable.tar.gz

tar -xvf libsodium-1.0.18-stable.tar.gz libsodium-stable

cd libsodium-stable/

./configure 

make -j

make -j check

sudo make install
```

c) Install pynacl and docker-compose

```
SODIUM_INSTALL=system pip3 install pynacl

SODIUM_INSTALL=system pip3 install docker-compose

docker-compose -v
```


### Golang

The installation of this component is fairly similar to its counterpart on x86 architecture. The newest release of Golang can be used.

```
wget https://golang.org/dl/go1.17.2.linux-arm64.tar.gz

sudo tar -C /usr/local -xzf go1.17.2.linux-arm64.tar.gz 

go --version

export PATH=$PATH:/usr/local/go/bin
```

### Python 3.x

Usually there is no need to install Python 3.x because it comes preinstalled with Operating Systems which were tested in this guide.

## Downloading and preparing git repositories

The Hyperledger Fabric Docker images need to be built within a Go workspace which should be created after installing dependencies. To manually create workspace, do the following:

```
mkdir -p $HOME/golang

export GOPATH=$HOME/go
```

Check whether path has been added by executing $ echo $PATH. If the GOPATH has not been added open the `$ ~/.profile` or `$ ~/.bashrc` and add the `GOPATH`, then execute $ `source ~/.bashrc` to save the paths.

Verify Golang environment setup by checking `$ go env` and see if `GOPATH` and `GOROOT` have been setup.

To successfully complete the deployment of Hyperledger Fabric, it will be necessary to download the following git repositories:

- Create a single directory for all deployments:

```
mkdir -p $HOME/go/src/github.com/hyperledger
```

- Go to the folder:

```
cd $HOME/go/src/github.com/hyperledger
```

- Clone the following repositories

```
git clone https://github.com/hyperledger/fabric-baseimage.git
git clone https://github.com/hyperledger/fabric.git
git clone https://github.com/hyperledger/fabric-ca.git
git clone https://github.com/hyperledger/fabric-chaincode-node
```
