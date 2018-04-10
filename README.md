# Docker Cheatsheat
You probably already know, but Docker is fantastic for:
- Development - No installation into your local environment, so no "it works on my machine conversations."
- Experimentation - You can easily try different languages and versions of different languages which is helpful for quickly determining if your code will run on the latest Python.
- Scaling - containers are made to scale up or down, with docker swarm, or other tools like Kubernetes or Rancher.
- Moving Legacy Apps to the Cloud - Getting your app running in a docker container means it will run in the cloud.
- Removing stubborn stains

## Slow Volumes with OS X
Docker works pretty well on OS X, but volumes can be slow to reflect filesystem changes without some tweaking.

*Note: this happens with pre 2011 Macs ie. Docker Toolbox + VirtualBox and newer Macs running Docker for Mac and XHyve.*

TL;DR INotify events move slowly host -> guest, so we add a container that syncs using Unison or Rsync.

https://github.com/docker/for-mac/issues/77  
[file-access-in-mounted-volumes-extremely-slow-cpu-bound](https://forums.docker.com/t/file-access-in-mounted-volumes-extremely-slow-cpu-bound/8076)

### docker-sync
Docker container with 2-way sync using Unison or 1-way using Rsync  
http://docker-sync.io/

#### Requirements
- Ruby

#### Installation
```
gem install docker-sync
```

Create a docker-sync.yml file
```
version: "2"
options:
  #verbose: true
  #max-attempt: 30
syncs:
  app-sync:                 # becomes the volume name in compose.yml
    src: './'               # the directory on the host system
    sync_host_ip: '192.168.99.100' # localhost for docker for mac
    sync_host_port: 10872   # unique port
    sync_strategy: 'unison' # or rsync
    sync_userid: '1000'     # id for www-data
```

#### Usage
```
# start the sync container
docker-sync start

# When stopping, ruby seems to barf with no "apparent" negative side-effects.  
# Also, it's a good idea to "clean" after stopping to remove the volume with the container
docker-sync stop
docker-sync clean


# When low on memory, docker-sync will have trouble syncing. This manifests itself as "copyconflict" files.  
# If you find copy conflicts everywhere, free up some memory  
# Hopefully you're working in your own branch (you're working in your own branch right?) and you can git reset --hard [previous commit sha]
```

#### Alternatives  
docker-osx-dev (1 way with Rsync)  
https://github.com/brikis98/docker-osx-dev  
https://www.ybrikman.com/writing/2015/05/19/docker-osx-dev/


## Common Flags
### Remove Container ( --rm )
```
# Removes a container after execution has stopped - no dangling containers
# Note: the --rm flag needs to come after "docker run" or "docker exec"

docker run --rm jamesway/scrapy
```

### Mount Volume ( -v )
```
# -v /full/host/path:/container/path
# to mount the current local directory to /app in container

docker run --rm -v $(pwd):/app jamesway/python36-django startproject myproject
```  

$(pwd) makes sense for my project's directory, but how do I know where to mount my project directory in the container?

1. Inspect the image's dockerfile for a WORK_DIR. Mounting to the WORK_DIR is probably what you want most of the time.
2. Specify a WORK_DIR in the container with the ( -w ) flag so that you can mount to it.

### work directory ( -w )
Specifes the work directory in the container.  

*If the dockerfile species no WORK_DIR, the default WORK_DIR is / (root) which can't be mounted with ( -v )*
```
# -w overrides the WORK_DIR specified in the image's dockerfile.

docker run --rm -w /src -v $(pwd):/src jamesway/php71-cli composer dump-autoload

# hello world in a different directory
mkdir hello && echo 'print ("hello world")' > hello/helloworld.py
docker run --rm  -w /code/hello -v $(pwd):/code python:3.6.4-alpine python helloworld.py

# or in PHP
mkdir hello && echo '<?php echo "hello world\n"; ?>' > hello/helloworld.php
docker run --rm  -w /app/hello -v $(pwd):/app php:7.1-cli-alpine php helloworld.php
```

### Interactive Terminal( -it )
If you want to pass input to container prompts, interact with an interpreter or shell into a container you need ( -it ).
```
# python interpreter
docker run --rm  -it python:3.6.4-alpine python

# passing commands to container prompts
docker run --rm -itv $(pwd):/app phpspec/phpspec run

# pass /bin/bash as the command to container to get bash
docker run --rm -it python:3.6-slim /bin/bash

# no bash in alpine images :(
docker run --rm  -it python:3.6.4-alpine python /bin/sh
```

## Commands
### Images
```
# list docker images
docker images

# remove an image
# you can get by with the first 3 characters of the image id, they're usually unique enough
docker rmi [image_id]

# build an image
# tag is optional and docker will add the default tag "latest"
docker build --rm -t image_name:tag dockerfile_dir

# eg:
docker build --rm -t scrapy-selenium-chrome:latest .
```

#### Building Images
- Shell In First - To identify what commands you need to run and in what order, you can log into the base image and run commands (apt-get/wget/curl) locally until you get the desired state. Make those commands your RUN statements
- Minimize RUNs - Each run command adds a layer to your image. Each layer adds size. Once the image your image is woking properly, minimize RUNs by chaining ( && ) them together
- Remove Build Dependencies - To slim down the image more, you can remove build time dependencies using "apt-get purge -y --auto-remove [packages]" in the same RUN statement you installed them.  
** A RUN layer is independent of other RUN layers. Installing packages in one RUN and removing them in another has NO effect on image size**
- Use Alpine Base Images - It's easier said than done, but Alpine images are super slim compared to Debian images. The downside is you will have to install and configure just about every aspect of your image manually.
-Sometimes you need you WORKDIR in your PATH
```
ENV APP_PATH /code
ENV PATH $APP_PATH:$PATH
WORKDIR $APP_PATH
```

eg: [https://hub.docker.com/r/jamesway/scrapy/~/dockerfile/]

### Containers
It's a good idea to run keep an eye on and remove dangling containers.
```
# list RUNNING containers
docker ps

# *Though you can remove a running container with (-f), you shouldn't remove a running container without stopping it first.*  
# *Listing the running containers gives you an idea if any container isn't running when it should be.*  
# *For instance, if mariadb isn't running, you can explicitly run that container (eg: docker-compose run mariadb) and the start messages should give you a quick idea of the problem.*  
# *Running a db in a docker is doable, but suggested for dev only as you'll probably need to purge the (named) volume(s).*

# list stopped containers
docker ps -a

# remove container
# again, the first 3 characters of the image id are usually unique enough
docker rm [container_id]

# removes volumes with container
# to keep things tidy, it's always a good idea to remove the volume if it doesn't need to persist
# a better option is to run the container with the --rm flag on the "run" or "exec" and the container and volume will be removed when the container is stopped.
docker rm -v [container_id]
```

### Volumes
It's a good idea to run keep an eye on and remove dangling volumes.
```
# list volumes
docker volume ls

# delete volumes
# again, the first 3 characters of the image id are usually unique enough
docker volume rm [volume_id]

# list dangling volumes (volumes without a container)
# This is here for completeness - I don't really use this too often, the one below is more useful.
docker volume ls -f dangling=true

# remove dangling volumes with a nested list dangling volumes
docker volume rm $(docker volume ls -q -f dangling=true)
```
