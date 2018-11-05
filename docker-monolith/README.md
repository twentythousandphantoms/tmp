# twentythousandphantoms_microservices
## Table of Contents
  
-  [Prepare Google Cloud project](#prepare-google-cloud-project)
-  [Create the docker host](#create-the-docker-host)
-  [Build my own docker image](#build-my-own-docker-image)
-  [Work with Docker Hub](#work-with-docker-hub)
- * [Prepare docker-host instances via Terraform](#prepare-docker-host-instances-via-terraform)
  
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
1. Copy configuration files to the container

   Add to Dockerfile:
   ```
   COPY mongod.conf /etc/mongod.conf
   COPY db_config /reddit/db_config
   COPY start.sh /start.sh 
   ```
1. Install the App dependencies and script file mode

   Add to Dockerfile:
   ```
   RUN cd /reddit && bundle install
   RUN chmod 0777 /start.sh
   ```
1. Start service after container starts

   Add to Dockerfile:
   ```
   CMD ["/start.sh"]
   ```
1. We are ready to build the image

   ```
   $ docker build -t reddit:latest . 
   Sending build context to Docker daemon 36.86kB
   Step 1/12 : FROM ubuntu:16.04
   16.04: Pulling from library/ubuntu
   ...
   Successfully built ad21718d3eb5
   Successfully tagged reddit:latest

   ```
   [Log example](https://raw.githubusercontent.com/express42/otus-snippets/master/hw-15/build.log)
   Option `-t` - Name and optionally a tag in the 'name:tag' format

1. Now we can start the container
  
   ```
   $ docker run --name reddit -d --network=host reddit:latest 
   ```
1. Allow INPUT TCP-traffic on port 9292

   ```
   $ gcloud compute firewall-rules create reddit-app \
    --allow tcp:9292 \
    --target-tags=docker-machine \
    --description="Allow PUMA connections" \
    --direction=INGRESS
   ```

1. Check

   ```
   $ docker-machine ls 
   NAME          ACTIVE   DRIVER   STATE     URL                       SWARM   DOCKER        ERRORS
   docker-host   *        google   Running   tcp://_your_IP_address__:2376           v18.06.1-ce 
   ```
   Open _your_IP_address_:9292 in browser

## Work with Docker Hub

>  Docker Hub is a cloud-based registry service which allows you to link to code repositories, build your images and test them, stores manually pushed images, and links to Docker Cloud so you can deploy images to your hosts.
1. Register in https://hub.docker.com/
1. Authentificate in Docker Hub

   ````
   docker login
   Login with your Docker ID to push and pull images from Docker Hub.
   If you don't have a Docker ID, head over to https://hub.docker.com to create one.

   Username: your-login
   Password:
 
   Login Succeeded
   ```
1. Uplaod the image to Docker Hub. So we can use it further

   ```
   $ docker tag reddit:latest <your-login>/otus-reddit:1.0

   $ docker push <your-login>/otus-reddit:1.0
   The push refers to a repository [docker.io/<your-login>/otus-reddit]
   c6e5100de1e0: Pushed
   ...
   a2022691bf95: Pushed
   1.0: digest:
   sha256:77c6070400a5b04f8db3f7c129a2c16084c2fcf186aa6b436c8d6f57e0014378 size:
   3448
   ```
1. Since that image is present in Docker Hub, we can run it not only with docker-host on GCP, but alsa on localhost or any other host.

   To check run it on other console:

   `$ docker run --name reddit -d -p 9292:9292 <your-login>/otus-reddit:1.0`
1. In additioanal you can discover the containers logs, log in running container, check the proccess list, stop the container, run it again, stop and remove, then start container without starting the app and check the proccesses with followed commands:

   ```
   • docker logs reddit -f
   • docker exec -it reddit bash
     • ps aux
     • killall5 1
   • docker start reddit
   • docker stop reddit && docker rm reddit
   • docker run --name reddit --rm -it <your-login>/otus-reddit:1.0 bash
     • ps aux
     • exit
   ```
1. And see the detailed information about the image, display specific fragment of information, run the app and add/remove directories and check the diff, make sure that after stopping and removing the container there is nothing changes with followed commands:

   ```
   • docker inspect <your-login>/otus-reddit:1.0
   • docker inspect <your-login>/otus-reddit:1.0 -f '{{.ContainerConfig.Cmd}}'
   • docker run --name reddit -d -p 9292:9292 <your-login>/otus-reddit:1.0
   • docker exec -it reddit bash
     • mkdir /test1234
     • touch /test1234/testfile
     • rmdir /opt
     • exit
   • docker diff reddit
   • docker stop reddit && docker rm reddit
   • docker run --name reddit --rm -it <your-login>/otus-reddit:1.0 bash
     • ls /
   ```
## Prepare docker-host instances via Terraform

1. Let use Terraform for creating docker-host instances in GCP. There is meta-parameter `count` available - The number of identical resources to create.

   ```
   resource "google_compute_instance" "docker-host" {
     count        = "${var.count}"
     name         = "docker-host-${count.index}"
   [...]
   ```
   Files:
     - docker-monolith/infra/tarreform/main.tf
     - docker-monolith/infra/tarreform/variables.tf
     - docker-monolith/infra/tarreform/terraform.tfvars

1. You can let the docker-machine know about existing hosts with `--google-use-existing` parameter

   ```
   $ docker-machine create --driver google \
     --google-use-existing --google-zone europe-west1-c \
     docker-host-0
   ```

## Create Ansible playbooks that installs Docker and runs App image


1. First, configure Ansible dynamic inventory to work with GCP.

   Install requirenments:

   ```
   $ echo "requests>=2.5.1\ngoogle-auth" > docker-monolith/infra/ansible/requirements.txt
   $ pip install -r docker-monolith/infra/ansible/requirements.txt
   ```
   
   2. [Create service-account](https://console.cloud.google.com/iam-admin/serviceaccounts) with "Project -> Editor" role and downloads a file that contains the private key. Store the file securely because this key cannot be recovered if lost.

   3. Enable "gcp_compute" [inventory plugin](https://docs.ansible.com/ansible/2.7/plugins/inventory.html):

   ```
   $ echo "[inventory]\nenable_plugins = gcp_compute, auto" >> docker-monolith/infra/ansible/ansible.cfg
   ```

   4. Create gcp_config. Docs: https://docs.ansible.com/ansible/2.7/plugins/inventory/gcp_compute.html. Example:

   ```
   plugin: gcp_compute
   projects:
     - project-111111
   filters:
   auth_kind: serviceaccount
   service_account_file: /home/111111/study/project-f7d9e118c0c9.json 
   hostnames:
     - name
   compose:
     ansible_host: networkInterfaces[0].accessConfigs[0].natIP
   ```

   5. Check:

   ```
   $ cd docker-monolith/infra/ansible/
   $ ansible-inventory -i project.gcp.yml
   ```
