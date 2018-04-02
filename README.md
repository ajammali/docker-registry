# Project Docker Registry

version: 1.0


## Table of Contents
* [Introduction](#introduction)
* [First step](#first-step)
* [Second step](#second-step)
* [Third step](#third-step)
* [Fourth step](#development)
  * [Storage Data](#build)
  * [Ca-Certs](#deploy)
  * [Proxy setup](#vertx-applications)
  * [Load Registry Base Image](#using-docker)
* [Final step](#gateways)

## Introduction:
Welcome to the Docker Registry project. This README explains how to setup and launch an unsecured docker registry server on Centos7 machine.

The VM configuration part (two first steps) is not automated yet, so it must be executed manually. The remaining steps can be achieved by just running the deploy-server playbook.

Follow these steps above and if lucky you will have a working registry server ;)

There are 5 main steps which must be accomplished carefully:

- Make sure your future remote server is running under Centos7.
- Make sure the user on your local machine has its public ssh key deployed on the remote server.
- Install the proper packages.
- Setup docker registry
- Deploy the registry server

You can simply run the playbook *deploy-server.yml* after adding the proper inventory, like this:

```sh
ansible-playbook -i inventories/local.ini deploy-server.yml --skip-tags "with_cacerts,wih_proxy"
```
To destroy your registry server, and reset all the customized setup:

```sh
ansible-playbook -i inventories/local.ini deploy-server.yml --tags uninstall
```

## First step: _**Install Vagrant Virtualbox-based Machine**_

Install Centos 7.2/7.4 on your machine.

Since I am using a pre-configured virtual machines, no extra action is required to be executed on your target machine.

For those who want to use centos/7 vagrant box, they will need to do some extra work to get it done.

```sh
cd /docker-registry/vagrant/

vagrant init centos/7 && vagrant up
```
This command will install and launch the latest centos7 vagrant box. 

After creating the virtual centos machine, you will need to add the new machine to your inventory file under /inventories/.

the local PORT used for this machine will be shown on the debug information.

## Second step: _**Setup the ansible user**_

The user *vagrant* is the default installed superuser. You can use it to create your own user *ansible*.

```sh
vagrant sh
sudo useradd ansible
sudo passwd ansible 
```

Once the user is added, you must remotely install your local ssh-key on your remote machine, using this command:

```sh
ssh-copy-id -p PORT ansible@localhost
```
A little bit of extra work is needed to be done here: 
- ssh-connect to your centos VM.
- add your ansible user to the sudoers file:
```sh
sudo visudo

Add this line: ansible ALL=(ALL) NOPASSWD: ALL
```
- edit the sshd config file:
```sh
sudo vi /etc/ssh/sshd_config

then enable PasswordAuthentication by setting it to yes
```
- restart sshd
```sh 
sudo systemctl restart sshd
```

*PORT* is the default local port used by your vagrant box, usually it's *2222* if it is the first running machine on your local environment.


## Third step: _**Install Packages**_

This list of packages need to be installed on your remote machine:

```sh
    - lvm2
    - device-mapper
    - device-mapper-persistent-data
    - device-mapper-event
    - device-mapper-libs
    - device-mapper-event-libs
    - container_selinux
    - libcgroup
    - libtool
    - docker-ce
```

Usually these packages are installed automatically by just installing *docker-ce* package. For that purpose, the docker-ce.repo must be added to the yum repositories list.

If your remote machine is behind a proxy server then you will have to setup the proxy configuration files. Or simply you can install all of these RPM packages manually.

## Fourth step: _**Registry Setup**_

_Storage Data_ :

Create a new filesystem repository that will be used to store and manage containers running under registry server.

 ```sh
 sudo mkdir /data/docker-registry
 ```
After creating the new filesystem, you will need to do this: 

- Copy all setup folder from */var/lib/docker/* to the registry setup location.
- Delete */var/lib/docker/* folder and replace it with a symlink with the same name that points to the new filesystem directory.
- Add a new symlink under */var/lib/registry/* that points to the new filesystem directory. 
 
This new storage directory should be specified on the start command like this: 
 
 ```sh
 ... -e STORAGE_PATH=/data/docker-registry ...
 ```
 
_Ca-Certs_ :
 
Add a new repository */opt/docker/certs/* to deploy all registry authorization certificates.
 
_Configuration File_ :
  
As default behaviour, docker registry uses its owen configuration file.

This file that we setup, must be specified on the start command as follows: 

 ```sh
 ... -v `pwd`/config.yml:/etc/docker/registry/config.yml
 ```
 
A later version of this project will use the adjusted configuration file and will explain its different parts.

_Proxy setup_ :

The proxy setup is not useful if your remote registry server does not use it.

Create a new repository */etc/systemd/system/docker.service.d/* then put the *http-proxy.conf* and the *https-proxy.conf* under it.

_Load Registry Base Image_ :

Docker Registry is a server based on a running container using an image called *registry*.

This image need to be loaded to the docker server in order to use it in creating our registry container.

 ```Gatewayssh
 docker pull registry:{{ version }}
 ```
 
On our registry server we are using *registry:2* image to build our registry container.

## Final step: _**Deploy Registry**_

The registry must run in a daemon mode.

All environment variables and configuration files and directories are specified on the running command.

- Running in secure mode (using ca-certs)
 ```sh
 docker run -d -p 443:443 --name registry -e STORAGE_PATH=/docker-registry -v /data/docker-registry:/docker-registry -v /opt/docker/certs/:/opt/docker/certs/ -e REGISTRY_HTTP_ADDR=0.0.0.0:443 -e REGISTRY_HTTP_TLS_CERTIFICATE=/opt/docker/certs/ca.crt -e REGISTRY_HTTP_TLS_KEY=/opt/docker/certs/ca.key registry:2
 ```
 
- Running in insecure mode (without ca-certs)
 ```sh
 docker run -d -p 5000:5000 --name registry -e STORAGE_PATH=/data/docker-registry -v /data/docker-registry:/data/docker-registry registry:2"
 ```
 
To test if you registry is accessible from inside the remote machine:
```sh
 curl -I http://localhost:5000/v2/
 
 HTTP/1.1 200 OK
 Content-Length: 2
 Content-Type: application/json; charset=utf-8
 Docker-Distribution-Api-Version: registry/2.0
 X-Content-Type-Options: nosniff
 Date: Mon, 02 Apr 2018 14:28:54 GMT

 ```
 
 On the remote mode: 
 ```sh
  curl -I http://registry.domain.name.or.ip.adress:5000/v2/
  
  HTTP/1.1 200 OK
  Content-Length: 2
  Content-Type: application/json; charset=utf-8
  Docker-Distribution-Api-Version: registry/2.0
  X-Content-Type-Options: nosniff
  Date: Mon, 02 Apr 2018 14:28:54 GMT
 
  ```
 
 To test your work, you can push/pull images from the registry
 - To push an image you must tag it under the registry domain name:
 ```sh 
 docker tag myImage:version registry.domain.name:5000/myImage
 ```
 - After tagging it, you can push it to the registry like this: 
 ```sh 
 docker push registry.domain.name:5000/myImage:version
 ```
 - To pull an image from your registry: 
 ```sh 
 docker pull docker push registry.domain.name:5000/myImage:version
 ```
 
 ```sh 
 [ansible@localhost ~]$ sudo docker push localhost:5000/ubuntu:16.04
 The push refers to repository [localhost:5000/ubuntu]
 db584c622b50: Pushed 
 52a7ea2bb533: Pushed 
 52f389ea437e: Pushed 
 88888b9b1b5b: Pushed 
 a94e0d5a7c40: Pushed 
 16.04: digest: sha256:52286464db54577a128fa1b1aa3c115bd86721b490ff4cbd0cd14d190b66c570 size: 1357
 [ansible@localhost ~]$ 

 ```