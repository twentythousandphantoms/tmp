## Table of Contents
  
- [Prepare Google Cloud project](#prepare-google-cloud-project)
- [Create the docker host](#create-the-docker-host)
- [Build my own docker image](#build-my-own-docker-image)
- [Work with Docker Hub](work-with-docker-hub)
  
## prepare Google Cloud project
  
1. Create new project named "docker": https://console.cloud.google.com/compute
1. Remember created Project ID
1. Install GCloud SDK: https://cloud.google.com/sdk/
1. Initialize gcloud:

   `gcloud init` - The browser should open. Grant access rights. Choose project.
1. Get credentials file that will be used by docker-machine:

   `gcloud auth application-default login`
1. Install `docker-machine` on Linux: https://docs.docker.com/machine/install-machine/ 

   You can use Docker Machine to create Docker hosts on your local Mac or Windows box, on your company network, in your data center, or on cloud providers like Azure, AWS, or Digital Ocean.

## Create the docker-host

1. Create

   ```
   $ export GOOGLE_PROJECT=_your_project_
   $ docker-machine create --driver google \
    --google-machine-image https://www.googleapis.com/compute/v1/
   projects/ubuntu-os-cloud/global/images/family/ubuntu-1604-lts \
    --google-machine-type n1-standard-1 \
    --google-zone europe-west1-b \
    docker-host
   ```
1. Check

   ```
   $ docker-machine ls
   NAME          ACTIVE   DRIVER   STATE     URL                       SWARM   DOCKER        ERRORS
   docker-host   -        google   Running   tcp://35.238.43.62:2376           v18.06.1-ce   

   ```
1. 
   `$ eval $(docker-machine env docker-host)`

### docker-machine

- Create
   `docker-machine create <name>`
- Switch
   `eval $(docker-machine env <name>).`
- Switch to local docker
   `eval $(docker-machine env --unset)`
- Delete
   `docker-machine rm <name>`

## Build my own docker image

1. For the futher work we need four files: `mongod.conf` `db_config` `start.sh` and `Dockerfile`. 
1. Lets create App Image

   Dockerfile:
   ```
   FROM ubuntu:16.04
   ```
1. There are mongo and ruby needed for this app. Update repo cache and install the packets

   Add to Dockerfile:
   ```
   RUN apt-get update
   RUN apt-get install -y mongodb-server ruby-full ruby-dev build-essential git
   RUN gem install bundler
   ```
1. Download the App to the container

   Add to Dockerfile:
   ```
   RUN git clone -b monolith https://github.com/express42/reddit.git 
   ```

