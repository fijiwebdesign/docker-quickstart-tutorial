# Docker quickstart

**Consise version of quickstart from https://docs.docker.com/get-started/**

Docker quickstart on how to run a container with python and redis using `Dockerfile`, create an image and share it to docker repo, generate local vms to run the app load-balanced using `docker-compose.yml`, create a distributed swarm of remote machines to run the app load-balanced.

## Docker commands cheatsheet 

```
docker build -t friendlyhello .  # Create image using this directory's Dockerfile
docker run -p 4000:80 friendlyhello  # Run "friendlyname" mapping port 4000 to 80
docker run -d -p 4000:80 friendlyhello         # Same thing, but in detached mode
docker container ls                                # List all running containers
docker container ls -a             # List all containers, even those not running
docker container stop <hash>           # Gracefully stop the specified container
docker container kill <hash>         # Force shutdown of the specified container
docker container rm <hash>        # Remove specified container from this machine
docker container rm $(docker container ls -a -q)         # Remove all containers
docker image ls -a                             # List all images on this machine
docker image rm <image id>            # Remove specified image from this machine
docker image rm $(docker image ls -a -q)   # Remove all images from this machine
docker login             # Log in this CLI session using your Docker credentials
docker tag <image> username/repository:tag  # Tag <image> for upload to registry
docker push username/repository:tag            # Upload tagged image to registry
docker run username/repository:tag      
```

## `Dockerfile`

The `Dockerfile` specfies how to build the docker image.

```
# Use an official Python runtime as a parent image
FROM python:2.7-slim

# Set the working directory to /app
WORKDIR /app

# Copy the current directory contents into the container at /app
COPY . /app

# Install any needed packages specified in requirements.txt
RUN pip install --trusted-host pypi.python.org -r requirements.txt

# Make port 80 available to the world outside this container
EXPOSE 80

# Define environment variable
ENV NAME World

# Run app.py when the container launches
CMD ["python", "app.py"]
```

## Create image and publish to repo

Create the image

```
docker build -t friendlyhello .
```

Tag image

```
docker tag friendlyhello seedess/python-quickstart:hello
```

Push image to repo

```
docker login
docker push seedess/python-quickstart:hello
```

Run image from repo (any machine with access)

```
docker run seedess/python-quickstart:hello
```

## `docker-compose.yml`

The `docker-compose.yml` file specifies how to run a containter image in production 

```
version: "3"
services:
  web:
    # replaced username/repo:tag 
    image: seedess/python-quickstart:hello
    deploy:
      replicas: 5
      resources:
        limits:
          cpus: "0.1"
          memory: 50M
      restart_policy:
        condition: on-failure
    ports:
      - "4000:80"
    networks:
      - webnet
networks:
  webnet:
```

### This `docker-compose.yml` file tells Docker to do the following:

Pull the image we uploaded in step 2 from the registry.

Run 5 instances of that image as a service called web, limiting each one to use, at most, 10% of the CPU (across all cores), and 50MB of RAM.

Immediately restart containers if one fails.

Map port 4000 on the host to web’s port 80.

Instruct web’s containers to share port 80 via a load-balanced network called webnet. (Internally, the containers themselves publish to web’s port 80 at an ephemeral port.)

Define the webnet network with the default settings (which is a load-balanced overlay network).

## Run app load balanced

```
docker swarm init
docker stack deploy -c docker-compose.yml getstartedlab
docker service ls # find service id, name
docker service ps getstartedlab_web # list tasks for service
docker container ls -q # show all tasks
docker stack rm getstartedlab # take down app
docker swarm leave --force # take down swarm
```

## App cheatsheet 

```
docker stack ls                                            # List stacks or apps
docker stack deploy -c <composefile> <appname>  # Run the specified Compose file
docker service ls                 # List running services associated with an app
docker service ps <service>                  # List tasks associated with an app
docker inspect <task or container>                   # Inspect task or container
docker container ls -q                                      # List container IDs
docker stack rm <appname>                             # Tear down an application
docker swarm leave --force      # Take down a single node swarm from the manager
```

## Set up your swarm

Now, create a couple of VMs using docker-machine, using the VirtualBox driver:

```
docker-machine create --driver virtualbox myvm1
docker-machine create --driver virtualbox myvm2
```

List vms

```
docker-machine ls
```

List vms output
```
NAME    ACTIVE   DRIVER       STATE     URL                         SWARM   DOCKER        ERRORS
myvm1   -        virtualbox   Running   tcp://192.168.99.100:2376           v18.06.1-ce
myvm2   -        virtualbox   Running   tcp://192.168.99.101:2376           v18.06.1-ce
```

## Initialize swarm 

Make `myvm1` swarm manager by sending it `docker swarm init` via ssh

```
docker-machine ssh myvm1 "docker swarm init --advertise-addr 192.168.99.100"
```

Swarm manager init output

```
Swarm initialized: current node (kms5ml3kuiaywwrxir67k2uni) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-2eghmt05766swvivzefarr3ttxys8dtk5ynvxxay2k1yrckk3n-4tnweq07zpa0kowt5xseeeq8i 192.168.99.100:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

Add `myvm2` to swarm

```
docker-machine ssh myvm2 "docker swarm join --token SWMTKN-1-2eghmt05766swvivzefarr3ttxys8dtk5ynvxxay2k1yrckk3n-4tnweq07zpa0kowt5xseeeq8i 192.168.99.100:2377"
```

View nodes in swarm managed by `myvm1` by sending it `docker node ls`

```
docker-machine ssh myvm1 "docker node ls"
```

To leave swarm `docker swarm leave` on each node.

## Deploy your app on the swarm cluster

Configure a docker-machine shell to the swarm manager

```
docker-machine env myvm1
```

Configure output

```
export DOCKER_TLS_VERIFY="1"
export DOCKER_HOST="tcp://192.168.99.100:2376"
export DOCKER_CERT_PATH="/Users/chandra/.docker/machine/machines/myvm1"
export DOCKER_MACHINE_NAME="myvm1"
# Run this command to configure your shell:
# eval $(docker-machine env myvm1)
```

Run the configure command (system dependent)

```
eval $(docker-machine env myvm1)
```

Run `docker-machine ls` to verify that `myvm1` is now the active machine, as indicated by the `*` next to it.

```
docker-machine ls
```

`docker-machine ls` output

```
NAME    ACTIVE   DRIVER       STATE     URL                         SWARM   DOCKER        ERRORS
myvm1   *        virtualbox   Running   tcp://192.168.99.100:2376           v18.06.1-ce
myvm2   -        virtualbox   Running   tcp://192.168.99.101:2376           v18.06.1-ce
```

```
docker stack deploy -c docker-compose.yml getstartedlab
```