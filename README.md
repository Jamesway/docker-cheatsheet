# Docker OS X Cheatsheat
Docker works pretty well on OS X, but volumes can be slow to reflect filesystem changes without some tweaking.

### Slow Volumes with OS X

*Note: this happens with pre 2011 Macs ie. Docker Toolbox + VirtualBox and newer Macs running Docker for Mac and XHyve.*

TL;DR INotify events don't move host -> guest on OS X, so we add a container that syncs using Unison or Rsync.

https://github.com/docker/for-mac/issues/77
https://forums.docker.com/t/file-access-in-mounted-volumes-extremely-slow-cpu-bound/8076

### docker-sync

2-way sync with Unison or 1-way with Rsync, requires ruby
http://docker-sync.io/

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
  app-sync: # becomes the volume name in compose.yml
    src: './' # the directory on the host system
    sync_host_ip: '192.168.99.100' # localhost for docker for mac
    sync_host_port: 10872 # unique port
    sync_strategy: 'unison' #or rsync
    sync_userid: '1000' #id for www-data
```

```
docker-sync start
```

*Note: when stopping docker-sync, ruby seems to barf with no "apparent" negative side-effects. Also, it's a good idea to "clean" after stopping to remove the lingering volume(s)/container(s).*
```
docker-sync stop
docker-sync clean
```

*Note: When low on memory, docker-sync will have trouble syncing. This manifests itself with copyconflict files. If you find copy conflicts everywhere, free up some memory and if you're working in your own branch (you're working in your own branch right?), git reset --hard [previous commit sha] *

TODO look into docker-osx-dev (1 way with Rsync)
https://github.com/brikis98/docker-osx-dev
https://www.ybrikman.com/writing/2015/05/19/docker-osx-dev/


### Running Container Images

## Flags
**remove container flag ( --rm )**
Removes a container after execution has stopped - no dangling containers
*Note: the --rm flag needs to come after "docker run" or "docker exec"

**volume flag ( -v /full/path:/container/path )**
eg. mount the current local directory to /app in container
```
docker run --rm -v $(pwd):/app composer dump-autoload


```
*Note: inspect the image's dockerfile to see if a work directory is specified. If so, mount the current dir to it and there's no need to specify the work dir (-w).*

**interactive terminal flags ( -it )**
eg. python interpreter or testing suite or getting bash to poke around in a container
```
docker run --rm  -it python:3.6.4-alpine python

or

docker run --rm -itv $(pwd):/app phpspec/phpspec run

or

docker run --rm -it python:3.6-slim /bin/bash
```
*Note: when rolling your own image, ( -it ) with /bin/bash makes it easier to figure out package dependencies.*
*Note: when using an alpine based image, user ( -it ) with /bin/sh*

**work directory - specifies the directory where...the work is done**
The default work dir is root ( / ) and you can't mount to the container's root directory, so if no work dir is defined in the image and you need to mount a volume, you need to define a work dir.
```
echo 'print ("hello world")' > helloworld.py
docker run --rm  -v $(pwd):/code -w /code python:3.6.4-alpine python helloworld.py

or  

echo '<?php echo "hello world"; ?>' > helloworld.php
docker run --rm  -v $(pwd):/app -w /app php:7.1-cli-alpine php helloworld.php
```

### Commands

list docker images and remove
```
docker images

docker rmi [image id] #the first 3 characters are usually unique enough
```

list RUNNING containers
```docker ps```

*Note: You can't/shouldn't remove a running container without stopping it, but listing the running containers gives you an idea if any container isn't running when it should be. For instance, if mariadb isn't running, you can explicitly run that container - ```docker-compose run mariadb``` - and the start messages will display suggesting it might be a good idea to purge volumes.*

list stopped containers and remove
```
docker ps -a

docker rm [container id] #the first 3 characters are usually unique enough

docker rm -v [container id] #removes volumes with container
```

build an image
```
docker build --rm -t image_name:tag dockerfile_dir

docker build --rm -t scrapy-selenium-chrome:latest .
```

To list volumes

```docker volume ls```

To delete volumes

```docker volume rm [volume id]```

To list and remove dangling volumes that have no containers
```
docker volume ls -f dangling=true

docker volume rm $(docker volume ls -q -f dangling=true)
```
*Note: to prevent dangling volumes use [-v] when removing containers - docker rm -v [container id]*

### Maintenance
It's a good idea to to keep an eye on and remove dangling containers and dangling volumes
