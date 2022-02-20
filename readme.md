# Frontend Master Complete Intro to Containers

## Pull Docker Image

We'll be using this docker image for this course, so we might as well pull this
image before we start the course to save some time:
```sh
docker pull ubuntu:bionic
docker pull node:12-stretch
docker pull node:12-alpine
docker pull nginx:1.17
docker pull mongo:3
docker pull jguyomard/hugo-builder:0.55
```

## Why We Need Containers?

Instead of running the whole operating system through virtual machine, we can
just provide the environment we need and then tell containers to execute our
code.

Container use chroot, namespace, and cgroup (control group) to separate a group
of processes from each other.

The benefit of using container is we don't need to manage a separate operating
system like when we use virtual machine. If we use virtual machine, we need to
update or patch the operating system inside of virtual machine.

## Chroot

A container is kinda three different kernel features put together. The three
different features is chroot (change root), namespaces, cgroup (control
group).

There's no one idea of single container, it's actually those three features put
together and allow us to make a container.

So now, let's start with the first and kind of most obvious problem, that if we
have two user running processes on the same host operating system, how can we
make sure that we can't see each other's files? We want to make sure that no
one can else can see into the container and we don't want anyone inside of any
other container to be able to see out to other ones. That's what we're gonna do
with chroot (change root).

> We can use `cat /etc/issue` to check which distribution we're currently in
> the container.

### Example

Let's say we want to change root in one of our directory. First, we need to
make those directory, let's call it `my-new-root`. We can create `my-new-root`
directory with this command:
```sh
mkdir my-new-root
```

And then, we need to make a "copy" of operating system inside of `my-new-root`
because once we're inside of `my-new-root`, we won't be able to access command
outside of `my-new-root`.

And one of the things we want to copy is shell. Let's say we want to use `bash`
as our shell, we need to copy `/bin/bash` to `my-new-root` directory. We also
need to copy the `bash` dependencies, we can check `bash` dependencies using
this command:
```sh
ldd /bin/bash
```
which the output would be similar to this (this is the output on arch linux):
```sh
        linux-vdso.so.1 (0x00007ffe70793000)
        libreadline.so.8 => /usr/lib/libreadline.so.8 (0x00007f06d63df000)
        libdl.so.2 => /usr/lib/libdl.so.2 (0x00007f06d63d8000)
        libc.so.6 => /usr/lib/libc.so.6 (0x00007f06d620c000)
        libncursesw.so.6 => /usr/lib/libncursesw.so.6 (0x00007f06d6198000)
        /lib64/ld-linux-x86-64.so.2 => /usr/lib64/ld-linux-x86-64.so.2 (0x00007f06d654a000)
```

> The instructor said that we can ignore the one that doesn't have a path, in
> this case it's this file `linux-vdso.so.1`. Not sure why, tho.

We need to put `bash` and its dependencies in the same directory name, we can
do that by using this command:
```sh
cp --parent -v /bin/bash /usr/lib/libreadline.so.8 /usr/lib/libdl.so.2 /usr/lib/libc.so.6 /usr/lib/libncursesw.so.6 /lib64/ld-linux-x86-64.so.2 my-new-root/
```

After that, we can change root using this command:
```sh
# use sudo if you're non run this as root.
chroot my-new-root /bin/bash
```

> If we want to try this in docker, we can use this command:<br>
> `docker run -it --name archlinux-host --rm --privileged archlinux:latest`

## Namespaces

So far we've just been worrying about file system but we haven't been worrying
about other things. Let's say we have two chroot environment next to each
other, they can still see each other processes. So if person A and person B
also running in there, person A can kill the person B's processes.

That's what namespaces are. Namespaces are the idea of these various different
facets of linux that we can limit each one of these particular "jails"
processes to write or container environments. So, we can only see our processes
, networking layer, etc and we can't mess with anyone else's.

We can run a program in new namespace like this:
```sh
# for more info: `man unshare`.
# `chroot my-new-root /bin/bash` is the command we want to run with `unshare`
# command.
unshare --mount --uts --ipc --net --pid --fork --user --map-root-user chroot my-new-root /bin/bash
```

## Cgroups

Cgroups or control groups is basically to limit the physical resources (CPU,
memory, etc) we have on specific environment.

For more info, we can check [arch wiki about control groups](https://wiki.archlinux.org/title/cgroups).

## Docker Image

Pre-made container is called image, and we can base our image off of other
people's image and we kind of build these layers on top of each other.

Basically all images is just dumps out the state of a particular container
and ties it together.

We can export the current state of the running docker container with command
like this:
```sh
docker export -o docker-container.tar <docker-container-name>
```

### Docker Image Without Docker

After we export the docker container state into a docker image, we can access
those docker image using `chroot`, `unshare`, and `cgroups` manually. This
notes won't go into detail how to do that, if you're curious, you can check
the course website section [here](https://btholt.github.io/complete-intro-to-containers/docker-images-without-docker).

### Docker Image With Docker

We can run the docker image with this command:
```sh
# in this example, we use alphine docker image version 3.10
docker run --interactive --tty archlinux:latest

# or the shorter version, like this
docker run -it archlinux:latest
```

> This is easier than running the docker image manually (or without docker).

The order of docker command is critical, below is the breakdown:
```sh
# example of `<command>` is `run`.
# example of `<flags>` is `-it`.
# example of `<container-name>` is `archlinux`.
# example of `<tags>` is `latest`.
# example of `<command-that-run-inside-container>` is `bash`.
# if we don't put the `<command-that-run-inside-container>`, the container will
# run the default command it has.
docker <command> <flags> <container-name>:<tags> <command-that-run-inside-container>
```

> We can clean up unused docker image with this command `docker image prune`.

To check all the running docker, we can use this command:
```sh
docker ps
```

To make docker container running in the background, we can add `--detach` or
`-d` flags like this:
```sh
# run this
docker run --detach --interactive --tty archlinux:latest

# or this
docker run -dit archlinux:latest
```

To kill the docker container, we can use this command:
```sh
# `<container>` can be container name or id.
docker kill <container>
```
the command above just to stop the running container, not remove the container.
If we want to remove the container, we can use this command:
```sh
# `<container>` can be container name or id.
docker rm <container>
```
If we want to automatically remove the container after we kill the container
process, we can use `--rm` flags like this example:
```sh
# the placement of flags is not an issue as long as it's before the docker
# image.
docker run --name archlinux-host -dit --rm archlinux:latest
```

> Why would we want to remove the container? So that we can use the same
> container name. Choosing a name is not easy.

Even after we kill the running container, we can still see the logs of that
container (as long as we haven't remove the container) with this command:
```sh
# `<container>` can be container name or id.
docker logs <container>
```

## Tags

Tags is placed after the `:` next to the image like this:
```sh
docker run --name archlinux-host -dit --rm archlinux:<tags>
```
if we omitted the tags, it will default to `latest`. So when we use something
like this:
```sh
docker run --name archlinux-host -dit --rm archlinux
```
is equivalent to this:
```sh
docker run --name archlinux-host -dit --rm archlinux:latest
```

Usually when we're making our environment, we might want our container to
have specific version so that we can avoid dependencies issue on the latest
version.

Is there any pattern of how tags are made? Not really, it depends on the author
of the docker image.

> There's a kind of rule of thumb for choosing the docker image, pick the
> latest LTS for the linux that were used and the latest LTS for the app that
> we want to use (node, go, ruby, etc).

## Docker CLI

### Docker Pull

Pulling the docker image if it's not exist in the local machine. The command
is something like this:
```sh
docker pull archlinux
```

### Docker Run

Run or create a new container based on the image. The command is something like
this:
```sh
docker run -it --name archlinux-host archlinux
```

### Docker Exec

Execute a command on existing docker container. The command is something like
this:
```sh
docker exec archlinux-host ls
```

### Docker Inspect

Print out info about the docker image. The command is something like this:
```sh
docker inspect archlinux
```

### Docker Pause

Freeze all of its process tress and its just stop in its tracks. The command
is something like this:
```sh
docker pause archlinux-host
```

To unpause, we can use this command:
```sh
docker unpause archlinux-host
```

**We can't execute a command using `docker exec` if the container is paused**.

### Docker Kill

Kill the running docker container. The command is something like this:
```sh
docker kill archlinux-host
```

A simple trick to kill all the running container:
```sh
# `docker ps -q` will print out the docker container id.
docker kill $(docker ps -q)
```

### Docker History

Show the history of a docker image. The command is something like this:
```sh
docker history archlinux
```

### Docker Info

Show system-wide information of the host computer. The command is something
like this:
```sh
docker info
```

### Docker Top

Show the running process inside of docker container. The command is something
like this:
```sh
# `archlinux-host` is the container name, we can also use the id instead of
# name.
docker top archlinux-host
```

### Docker Ps

List the docker container. The command is something like this:
```sh
docker ps
```

If we want to see all the container, not only the running container, we can
use this command:
```sh
docker ps -a
```
usually this happen when we create container using `docker run` but forgot
to delete them using `docker rm`.

### Docker Logs

Fetch the logs of the docker container. The command is something like this:
```sh
docker logs archlinux-host
```

### Docker Rmi

Remove one or more docker images. Basically an alias for `docker image rm`.
The command is something like this:
```sh
docker rmi archlinux
```

### Docker Container Prune

Remove all stopped container. Related to `docker ps -a` command. The command
is something like this:
```sh
docker container prune
```

### Docker Restart

Restart one or more docker containers. The command is something like this:
```sh
docker restart archlinux-host
```

In some case, some containers won't respond to restart signal so it waits ten
seconds and do hard restarts for us.

### Docker Search

Search the docker hub for image. The command is something llke this:
```sh
docker search archlinux
```

## Dockerfile

So far, what we've been focused on has been running an existing docker
image. Now, we'll learn how to build docker image.

There's a bunch of ways to build a docker image, and the most common
one is with `Dockerfile`. `Dockerfile` is a file that docker read to build
the docker image for us.

To make things simple, `Dockerfile` is a series of instructions. So, **the
order matter**.

### Example 1

In this example, we'll make the most basic node `Dockerfile`.

The first thing we need to start out is a base container, here's the example:
```sh
FROM node:12-stretch
```

And then, the next things is tell the container to do something whenever it
start up. Here's an example:
```sh
# we give an array as the input
# it execute the last one, so if we have multiple CMD, only the last one
# will be executed.
# the command below is equivalent to `node -e 'console.log("lol")'`
CMD ["node", "-e", "console.log(\"lol\")"]
```

So if we put both of those lines together, it will be something like this
inside the `Dockerfile`:
```dockerfile
FROM node:12-stretch

CMD ["node", "-e", "console.log(\"lol\")"]
```

We can build those docker container with this command:
```sh
# `.` means look the Dockerfile in this directory.
# `--tag` means create a name tag for the container, so we can use this name
# instead of the container id.
# this will use `latest` tag by default.
docker build --tag my-node-app .

# this will use `69` tag.
docker build --tag my-node-app:69 .
```

To check the docker image we've created, we can use this command:
```sh
docker image ls
```

And we can run the image we've just created with this command:
```sh
docker run my-node-app
```

### Example 2

Another example, let's say we want to make a simple node server. We can copy
the `index.js` into the docker image like this:
```sh
COPY index.js index.js
```
and then run the server like this:
```sh
# this is similar to `node index.js`.
CMD ["node", "index.js"]
```

If we put the `Dockerfile` together, that will look like this
```dockerfile
FROM node:12-stretch

COPY index.js index.js

CMD ["node", "index.js"]
```

We can run the rebuild the docker image using this command:
```sh
# `-t` is equivalent to `--tag`.
docker build -t my-node-app .
```

We can run the server using this command:
```sh
# `--init` will run with the module `tini` and it will proxy those process
# signal we give, such as `ctrl-c`.
docker run --init my-node-app
```

The problem now, is that we don't have access to the port inside of docker
container we just created (see the [namespaces section](#namespaces). So, what
we can do is give `--publish` or `-p` flags to expose the port like this:
```sh
docker run --init -p <port-in-host-machine>:<port-in-docker> my-node-app
```

### Run As User

In the two example above, we run all the command as root. If we
want to run the command as user, we can do something like this:
```sh
USER <the-available-user-in-container>
```

If we combine that with [example 2](#example-2) above, we can do something
like this (in node docker image, there's another user called `node`):
```dockerfile
FROM node:12-stretch

USER node

COPY --chown=node:node index.js index.js

CMD ["node", "index.js"]
```

> There's `COPY` and `ADD` for `Dockerfile`. The rule of thumb is that
> if we're doing anything in the network or we need anything related to zip
> or tar or something similar, we can use `ADD`. Other than that, use `COPY`.

### Work Directory

By default, the work directory for copying a file is in root directory. We can
change that with `WORKDIR`. Below is an example how to use `WORKDIR`:
```dockerfile
FROM node:12-stretch

USER node

WORKDIR /home/node/src

COPY --chown=node:node index.js index.js

CMD ["node", "index.js"]
```

### Add Dependencies to Node App

When we're dealing with node dependencies, what we need to do is install
the dependencies using `npm` or `yarn` which we can do with `RUN` like
example below:
```dockerfile
FROM node:12-stretch

USER node

RUN mkdir /home/node/src

WORKDIR /home/node/src

COPY --chown=node:node . .

RUN yarn

CMD ["node", "index.js"]
```

Why we need `RUN mkdir /home/node/src`? Because by default, it will create a
directory as `root` instead of `node` user. When using `RUN`, we run the
command as the current user. With that, hopefully we avoid permission error.

### Cache Layers

If there's no change, then docker won't run it again. For example, if we
change something inside of `index.js` then we'll run these command again
(from previous example):
```sh
COPY --chown=node:node . .

RUN yarn

CMD ["node", "index.js"]
```
why did it run again from `COPY --chown=node:node . .`? Because we change
something inside current directory, and the crucial part is that we will
doing `yarn install` again even tho we didn't change dependencies. That's kind
of how cache works with `Dockerfile`.

So, with that in mind, we can do something like this:
```dockerfile
FROM node:12-stretch

USER node

RUN mkdir /home/node/src

WORKDIR /home/node/src

COPY --chown=node:node package.json package.json

RUN yarn

COPY --chown=node:node . .

CMD ["node", "index.js"]
```

The downside of this approach is that, we won't be able to get the patch
of our dependencies because wee only run `yarn install` if there's any change
to `package.json` file. That can be a good thing or a bad thing depending on our
circumstances.

## Docker Ignore

We can ignore the file or directory we don't want to include when we're
building a docker image. The concept is similar to `.gitignore`.

We can make docker ignore by creating `.dockerignore` file and put the
same format as `.gitignore`.

## Make Simple Node Docker Container

We can make a simple node docker container with alpine linux like this:
```dockerfile
FROM alpine:3.10

RUN apk add --update nodejs yarn
RUN addgroup -S node && adduser -S node -G node
USER node

RUN mkdir /home/node/src
WORKDIR /home/node/src

COPY --chown=node:node package.json package.json
RUN yarn

COPY --chown=node:node . .

CMD ["node", "index.js"]
```

The benefit of using alpine linux is that it's smaller, so we don't have
to worry about storage when using alpine linux.

## Multi-stage Builds

Docker allows us to do what's called multi-stage builds, which is like
"build something first, copy the output into a new container, and go
from there".

Here's an example of multi-stage builds:
```dockerfile
# build stage
FROM node:12-stretch AS build
WORKDIR /build
COPY package.json package.json
RUN yarn
COPY . .

# runtime stage
FROM alpine:3.10
RUN apk add --update nodejs
RUN addgroup -S node && adduser -S node -G node
USER node

RUN mkdir /home/node/src
WORKDIR /home/node/src

COPY --from=build --chown=node:node /build .

CMD ["node", "index.js"]
```

## Features in Docker

### Mounts

So far we've been dealing with self-contained containers, it's basically
a container that runs, does something, and then disappears.

There's some cases when we don't want everything in the containers goes away,
one of them is database. We can scale up or scale down the containers but we
don't want the data inside the container to disappears. So, that's why we're
gonna get into things called *mounts*.

#### Bind Mounts

Bind mounts are like portal to our host computer, so we can say "ok here's
directory and this container can only see this directory". So anything that
changes in the container, shows up on the host computer, and anything that
changes in the host computer, shows up in the container.

We can use mount using `--mount` flags, for more info check `man docker-run`.

#### Volume Mounts

The difference between bind mounts and volumes is that bind mounts are
literally just files on your computer that you're exposing inside of the
container. A volume mounts is something that docker manages for you, it's
not exposed on the host computer.

> Docker be like "Ok, you've created this new volume, this new file system
> and i'll just keep it. Anytime you ask for it, i'll hand it back to you."

**It's kind of a rule of thumb if we have information that really just for the
containers and it's never meant for the host computer, use volume mounts**.

> Use `docker volume prune` can save us quite a few of space.

## Networking in Docker

We can check the docker network available with this command:
```sh
docker network ls
```
the output is something like this:
```sh
NETWORK ID     NAME                       DRIVER    SCOPE
5ec0d181fbb6   bridge                     bridge    local
adc126979ccb   host                       host      local
96de523a646a   none                       null      local
```

And in this part, we'll be working with `bridge` network. `bridge` network
is basically allows us to connect two containers with each other.

In general, the instructor didn't recommend that we use the default `bridge`
network (don't know why tho). So, we're gonna create our own `bridge` network.

Here's an example of how to create the `bridge` docker network:
```sh
# `app-net` is the network name.
docker network create --driver=bridge app-net
```

## Docker Compose

Docker compose allows us to say "here's the container, here's another
container and here's how they talk to each other. Now spin them all up for
me", and we do that with one configuration file.

Docker compose is more suitable for development environments, less so for
production environments.

Docker compose says on its documentation that it is suitable for production
environments if we have basically one host running all of our containers, which
for the most part, most people don't. We don't wanna be tied to one host, we
wanna be able to scale up or scale down and we can't do that with docker
compose.

> One of the most useful things of docker compose is for CI/CD.

Here's the example of docker compose:
```sh
# version of docker compose.
version: "3"

# the various of containers that the docker compose is going to spin up.
services:
  web:
    # where the Dockerfile is, where it needs to build it.
    build: .
    # ports that we want to expose, similar to `--publish` flags.
    ports:
      - "3000:3000"
    # how to connect our code to this.
    volumes:
      # expose everything in current directory inside /home/node/code
      # it's actually a bind mount.
      - .:/home/node/src
      # basically treat node_modules differently, don't bring node_modules.
      - /home/node/src/node_modules
    # web container needs to talk to db container, this acts like dependencies
    # which make the db container spin up first.
    links:
      - db
    environment:
      MONGO_CONNECTION_STRING: mongodb://db:27017

  db:
    # pull docker image for mongo with tag 3.
    image: mongo:3
```

And then we can run this command to run the docker compose:
```sh
# for more info check `docker-compose --help`
docker-compose up
```

## Kubernetes

> Kubernetes is often abbreviates k8s (k then 8 letters then s).

Kubernetes is useful if we have a lot of containers, a lot of different kind
of services, and all the various services have complex relationships between
each other.

With kubernetes we can define the security rules, it scales out
very well (from one container to 100 containers and then going back to one
container).

Kubernetes also very good at checking the health of our containers. So if we
have like 69 containers and one of them starts going bad, kubernetes can detect
that via health checks and say "shut that container down, and spin new one up,
put it back in its place".

### Master

When we spin up out kubernetes cluster of any variety, the first thing is it's
gonna have a `master` and that's kind of like a central control that controls
everything else in the service.

### Nodes

Not to be confused with node.js, nodes are the worker that actually gonna be
doing the work for us and they're the one that actually talked to master. So
one node can contain multiple containers or only one container. So technically
a node is just like a deploy target, it's relatively unimportant of what the
node is as long as it accepts containers.

### Pods

Pod is basically a deployment use of the unit that cannot be separated. Let's
say we had user service and password service and they can't really be separated
because they need to be deployed together, that would be considered a pod. That
not always the case, a lot of times our pod will only be one container, like it
might just be one service that we're running.

### Service

Service is a group of pods that make up one backend.

When we have pods and when we have these containers, they will be talking to
each others, and those kind of routing things will be dynamic because we'll be
spinning up pods and destroying pods and likely to be doing all sort of
different things, and at any given time we don't actually know how many
containers will be running at any given time. And that's when services become
useful.

### Deployment

A deployment is where we describe what the things that we would like to go out.
We can have multiple deployments, we can also rollback deployment.

## Open Container Initiative (OCI)

Open Container Initiative (OCI) is basically a non-docker containers. Linux
containers are not just docker.

In this course, there's two OCI that were introduced (which is not in this
notes, you should check out the course website instead):
- Buildah
- Podman

> Docker is also a part of OCI. So we can build containers with docker and
> execute it with buildah or podman and vice versa.

## References

- [Course website](https://btholt.github.io/complete-intro-to-containers/).
- [Course repo](https://github.com/btholt/complete-intro-to-containers).
- [Solution for course exercise](https://github.com/btholt/projects-for-complete-intro-to-containers).
- [Ldd manual page](https://man7.org/linux/man-pages/man1/ldd.1.html).
- [Use chroot command in ubuntu](https://www.howtogeek.com/441534/how-to-use-the-chroot-command-on-linux/).
- [Syntax for chroot](https://unix.stackexchange.com/a/128048).
- [Cgroups in arch linux](https://wiki.archlinux.org/title/cgroups).
- [Docker hub container registry](https://hub.docker.com/search?q=&type=image).
- [Equivalent of npm ci](https://stackoverflow.com/a/69944063).
- [Kompose website](https://kompose.io/).
