# Docker
This project is based on https://github.com/sidpalas/devops-directive-docker-course
https://courses.devopsdirective.com/docker-beginner-to-pro/lessons/00-introduction/01-main

# Project Description
This Docker project is divided into 12 modules:

1. What is a Docker Container Image and Container?

2. Docker Application Architecture

3. Docker Installation and Set Up

4. Using 3rd Party Container Images

5. Example Web Application

6. Building Container Images

7. Container Registries

8. Running Containers

9. Container Security

10. Interacting with Docker Objects

11. Development Workflow

12. Deploying Containers

## What is a Docker Container Image and Container?

A **Docker container image** is a lightweight, standalone, executable package of software that includes everything needed to run an application. This package contains:

* Underlying OS dependencies

* Runtime dependencies (e.g., Python runtime)

* Libraries (e.g., SQL Alchemy, FastAPI)

* Application code

A **container image** is like a class in object-oriented programming, while a **container** is an instantiation of that class. A container allows us to create one or more standardized copies that are the same every time.

## 2. Docker Application Architecture

It is useful to break down the various components within the Docker ecosystem.

Docker Desktop is an application you install on development systems that provides:

* A client application:

    * Command Line Interface (CLI): Interact with Docker using commands likedocker run or docker pull.
    * Graphical User Interface (GUI): Browse images, configure CPU, memory, and disk space allocation.
    * Credential Helper: Store credentials for private registries.
    * Extensions: Third-party software that provides additional functionality.
* A Linux virtual machine containing:
    * Docker daemon (dockerd): Manages container objects, networking, and volumes within the server host application. It exposes the Docker API for communication with the client application.
    * (Optional) Kubernetes cluster

## 3. Installation and Set up

### macOS Installation

1. Visit the Docker Docs: Go to docs.docker.com/get-docker and click on "Docker Desktop for Mac" for either Intel or Apple M1, depending on your system.
2. Download the installer: The installer file (.dmg) will begin downloading.
3. Install Docker Desktop: Once downloaded, open the .dmg file and drag Docker into the Applications folder.
4. Launch Docker Desktop: Go to your Applications and click on Docker to start the application.

### Run Simple Containers with Docker

**Example 1: Whalesay Container**
1. Run the Whalesay Container with the following command:

``` docker run docker/whalesay cowsay "Hey Team! 👋" ``` 

This command will pull the public whalesay image from Docker Hub, download it, and run a container with the custom command provided.

2. After the container finishes running, you should see an ASCII art output with the custom phrase "Hey Team! 👋".

```
 ________________
< Hey Team! 👋 >
 ----------------
    \
     \
      \
                    ##        .
              ## ## ##       ==
           ## ## ## ##      ===
       /""""""""""""""""___/ ===
  ~~~ {~~ ~~~~ ~~~ ~~~~ ~~ ~ /  ===- ~~~
       \______ o          __/
        \    \        __/
          \____\______/
```

**Example 2: Postgres Container**

1. Run the Postgres container: Execute the following command in your terminal or command prompt

``` docker run -e POSTGRES_PASSWORD=foobarbaz -p 5432:5432 postgres:15.1-alpine ```

This command will:

* Set the POSTGRES_PASSWORD environment variable (required for the container to start)
* Publish port 5432 on your localhost and connect it to port 5432 inside the container
* Pull the postgres:15.1-alpine image from Docker Hub, if it's not already available locally
* Start the container, creating a running Postgres 15.1 instance on the Alpine operating system

2. Verify the Postgres container is running: Open a PostgreSQL client such as pgAdmin and connect to the database by adding a new server with the following information:
    - Host: localhost
    - Port: 5432
    - User: postgres
    - Password: foobarbaz

3. Execute a query: Run a sample query to confirm the connection and access to the database inside the container. For example:

```SELECT * FROM information_schema.tables;```
This query will return all the tables from the information schema within the database running inside the container.

## 4. Using 3rd Party Containers

### A. Installing Dependencies
Let's experiment with how installing something into a container at runtime behaves!

```
# Create a container from the ubuntu image
docker run --interactive --tty --rm ubuntu:22.04

# Try to ping google.com
ping google.com -c 1 # This results in `bash: ping: command not found`

# Install ping
apt update
apt install iputils-ping --yes

ping google.com -c 1 # This time it succeeds!
exit
```

Let's try that again:
```
docker run -it --rm ubuntu:22.04
ping google.com -c 1 # It fails! 🤔
```

It fails the second time because we installed it into that read/write layer specific to the first container, and when we tried again it was a **separate** container with a **separate** read/write layer!

We can give the container a name so that we can tell docker to reuse it:
```
# Create a container from the ubuntu image (with a name and WITHOUT the --rm flag)
docker run -it --name my-ubuntu-container ubuntu:22.04

# Install & use ping
apt update
apt install iputils-ping --yes
ping google.com -c 1
exit

# List all containers
docker container ps -a | grep my-ubuntu-container
docker container inspect my-ubuntu-container

# Restart the container and attach to running shell
docker start my-ubuntu-container
docker attach my-ubuntu-container

# Test ping
ping google.com -c 1 # It should now succeed! 🎉
exit
```

We generally never want to rely on a container to persist the data, so for a dependency like this, we would want to include it in the image:

```bash
# Build a container image with ubuntu image as base and ping installed
docker build --tag my-ubuntu-image -<<EOF
FROM ubuntu:22.04
RUN apt update && apt install iputils-ping --yes
EOF

# Run a container based on that image
docker run -it --rm my-ubuntu-image

# Confirm that ping was pre-installed
ping google.com -c 1 # Success! 🥳
```
The `FROM... RUN...` stuff is part of what is called a `Dockerfile` that is used to specify how to build a container image. We will go much deeper into building containers later in the course, but for now just understand that for anything we need in the container at runtime we should build it into the image! 

The one exception to this rule is environment specific configuration (environment variables, config files, etc...) which can be provided at runtime as a part of the environment (see: https://12factor.net/config).

### B. Persisting Data Produced by the Application:

Often, our applications produce data that we need to safely persist (e.g. database data, user uploaded data, etc...) even if the containers are destroyed and recreated. Luckily, Docker (and containers more generally) have a feature to handle this use case called `Volumes` and `mounts`!

![](./readme-assets/volumes.jpg)

`Volumes` and `mounts` allow us to specify a location where data should persist beyond the lifecycle of a single container. The data can live in a location managed by Docker (`volume mount`), a location in your host filesystem (`bind mount`), or in memory (`tmpfs mount`, not pictured). 

Let's experiment with how creating some data within a container at runtime behaves!

```bash
# Create a container from the ubuntu image
docker run -it --rm ubuntu:22.04

# Make a directory and store a file in it
mkdir my-data
echo "Hello from the container!" > /my-data/hello.txt

# Confirm the file exists
cat my-data/hello.txt
exit
```

If we then create a new container, (as expected) the file does not exist!

```bash
# Create a container from the ubuntu image
docker run -it --rm ubuntu:22.04

# Check if the file exists
cat my-data/hello.txt # Produces error: `cat: my-data/hello.txt: No such file or directory`
```

#### i. Volume Mounts
We can use volumes and mounts to safely persist the data.


```bash
# create a named volume
docker volume create my-volume

# Create a container and mount the volume into the container filesystem
docker run  -it --rm --mount source=my-volume,destination=/my-data/ ubuntu:22.04
# There is a similar (but shorter) syntax using -v which accomplishes the same
docker run  -it --rm -v my-volume:/my-data ubuntu:22.04

# Now we can create and store the file into the location we mounted the volume
echo "Hello from the container!" > /my-data/hello.txt
cat my-data/hello.txt
exit
```

We can now create a new container and mount the existing volume to confirm the file persisted:

```bash
# Create a new container and mount the volume into the container filesystem
docker run  -it --rm --mount source=my-volume,destination=/my-data/ ubuntu:22.04
cat my-data/hello.txt # This time it succeeds! 
exit
```

Where is this data located? On linux it would be at `/var/lib/docker/volumes`... but remember, on docker desktop, Docker runs a linux virtual machine.

One way we can view the filesystem of that VM is to use a [container image](https://hub.docker.com/r/justincormack/nsenter1) created by `justincormat` that allows us to create a container within the namespace of PID 1. This effectively gives us a container with root access in that VM. 

#### ii. Bind Mounts

Alternatively, we can mount a directory from the host system using a bind mount:

```bash
# Create a container that mounts a directory from the host filesystem into the container
docker run  -it --rm --mount type=bind,source="${PWD}"/my-data,destination=/my-data ubuntu:22.04
# Again, there is a similar (but shorter) syntax using -v which accomplishes the same
docker run  -it --rm -v ${PWD}/my-data:/my-data ubuntu:22.04

echo "Hello from the container!" > /my-data/hello.txt

# You should also be able to see the hello.txt file on your host system
cat my-data/hello.txt
exit
```

Bind mounts can be nice if you want easy visibility into the data being stored, but there are a number of reasons outlined at https://docs.docker.com/storage/volumes/ (including speed if you are running Docker Desktop on windows/mac) for why volumes are preferred. 

## II. Use Cases

Now that we have an understanding of how data storage works with containers we can start to explore various use cases for running 3rd party containers.

### A. Databases

Databases are notoriously fickle to install and configure. For development, where you might need to run multiple versions of a single database or create a fresh database for testing purposes running in a container can be a massive improvement.

The setup/installation is handled by the container image, and all you need to provide is some configuration values. Switching between versions of the database is as easy as specifying a different image tag (e.g. `postgres:14.6` vs `postgres:15.1` ).

Here are a some useful databases container images and sample commands that attempt to mount the necessary data directories into volumes and set key environment variables.

#### Postgres 
https://hub.docker.com/_/postgres
```bash
docker run -d --rm \
  -v pgdata:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=foobarbaz \
  -p 5432:5432 \
  postgres:15.1-alpine

# With custom postresql.conf file
docker run -d --rm \
  -v pgdata:/var/lib/postgresql/data \
  -v ${PWD}/postgres.conf:/etc/postgresql/postgresql.conf \
  -e POSTGRES_PASSWORD=foobarbaz \
  -p 5432:5432 \
  postgres:15.1-alpine -c 'config_file=/etc/postgresql/postgresql.conf'
```

#### Mongo
https://hub.docker.com/_/mongo
```bash
docker run -d --rm \
  -v mongodata:/data/db \
  -e MONGO_INITDB_ROOT_USERNAME=root \
  -e MONGO_INITDB_ROOT_PASSWORD=foobarbaz \
  -p 27017:27017 \
  mongo:6.0.4

# With custom mongod.conf file
docker run -d --rm \
  -v mongodata:/data/db \
  -v ${PWD}/mongod.conf:/etc/mongod.conf \
  -e MONGO_INITDB_ROOT_USERNAME=root \
  -e MONGO_INITDB_ROOT_PASSWORD=foobarbaz \
  -p 27017:27017 \
  mongo:6.0.4 --config /etc/mongod.conf
```

#### Redis
https://hub.docker.com/_/redis

Depending how you are using redis within your application, you may or may not care if the data is persisted.

```bash
docker run -d --rm \
  -v redisdata:/data \
  redis:7.0.8-alpine

# With custom redis.conf file
docker run -d --rm \
  -v redisdata:/data \
  -v ${PWD}/redis.conf:/usr/local/etc/redis/redis.conf \
  redis:7.0.8-alpine redis-server /usr/local/etc/redis/redis.conf
```

#### MySQL
https://hub.docker.com/_/mysql
```bash
docker run -d --rm \
  -v mysqldata:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=foobarbaz \
  -p 3306:3306 \
  mysql:8.0.32

# With custom conf.d
docker run -d --rm \
  -v mysqldata:/var/lib/mysql \
  -v ${PWD}/conf.d:/etc/mysql/conf.d \
  -e MYSQL_ROOT_PASSWORD=foobarbaz \
  -p 3306:3306 \
  mysql:8.0.32
```

#### Elasticsearch
https://hub.docker.com/_/elasticsearch
```bash
docker run -d --rm \
  -v elasticsearchdata:/usr/share/elasticsearch/data
  -e ELASTIC_PASSWORD=foobarbaz \
  -e "discovery.type=single-node" \
  -p 9200:9200 \
  -p 9300:9300 \
  elasticsearch:8.6.0
```

#### Neo4j
https://hub.docker.com/_/neo4j

```bash
docker run -d --rm \
    -v=neo4jdata:/data \
    -e NEO4J_AUTH=neo4j/foobarbaz \
    -p 7474:7474 \
    -p 7687:7687 \
    neo4j:5.4.0-community
```

### B. Interactive Test Environments

#### i. Operating systems

```bash
# https://hub.docker.com/_/ubuntu
docker run -it --rm ubuntu:22.04

# https://hub.docker.com/_/debian
docker run -it --rm debian:bullseye-slim

# https://hub.docker.com/_/alpine
docker run -it --rm alpine:3.17.1

# https://hub.docker.com/_/busybox
docker run -it --rm busybox:1.36.0 # small image with lots of useful utilities
```


#### ii. Programming runtimes:
```bash
# https://hub.docker.com/_/python
docker run -it --rm python:3.11.1

# https://hub.docker.com/_/node
docker run -it --rm node:18.13.0

# https://hub.docker.com/_/php
docker run -it --rm php:8.1

# https://hub.docker.com/_/ruby
docker run -it --rm ruby:alpine3.17
```

### C. CLI Utilities

Sometimes you don't have a particular utility installed on your current system, or breaking changes between versions make it handy to be able to run a specific version of a utility inside of a container without having to install anything on the host!

**jq (json command line utility)**

https://hub.docker.com/r/stedolan/jq
```bash
docker run -i stedolan/jq <sample-data/test.json '.key_1 + .key_2'
```

**yq (yaml command line utility)**

https://hub.docker.com/r/mikefarah/yq
```bash
docker run -i mikefarah/yq <sample-data/test.yaml '.key_1 + .key_2'
```

**sed**

GNU `sed` behaves differently from the default MacOS version for certain edge cases.
```bash
docker run -i --rm busybox:1.36.0 sed 's/file./file!/g' <sample-data/test.txt
```

**base64**

GNU `base64` behaves differently from the default MacOS version for certain edge cases.
```bash
# Pipe input from previous command
echo "This string is just long enough to trigger a line break in GNU base64." | docker run -i --rm busybox:1.36.0 base64

# Read input from file
docker run -i --rm busybox:1.36.0 base64 </sample-data/test.txt
```

**Amazon Web Services CLI**

https://hub.docker.com/r/amazon/aws-cli
```bash
# Bind mount the credentials into the container
docker run --rm -v ~/.aws:/root/.aws amazon/aws-cli:2.9.18 s3 ls
```

**Google Cloud Platform CLI**

```bash
# Bind mount the credentials into the container
docker run --rm -v ~/.config/gcloud:/root/.config/gcloud gcr.io/google.com/cloudsdktool/google-cloud-cli:415.0.0 gsutil ls
# Why is the container image so big 😭?! 2.8GB
```

### D. Improving the Ergonomics

If you plan to use one of these utilities inside of a container frequently, it can be useful to use a shell function or alias to make the ergonomics feel like the program is installed on the host. Here are examples of this for `yq`:

```bash
# Shell function
yq-shell-function() {
  docker run --rm -i -v ${PWD}:/workdir mikefarah/yq "$@"
}
yq-shell-function <sample-data/test.yaml '.key_1 + .key_2'

---

# Alias
alias 'yq-alias=docker run --rm -i -v ${PWD}:/workdir mikefarah/yq'
yq-alias <sample-data/test.yaml '.key_1 + .key_2'
```

## 5. Sample Web Application

### Minimal 3 tier web application
- **React frontend:** Uses react query to load data from the two apis and display the result
- **Node JS and Golang APIs:** Both have `/` and `/ping` endpoints. `/` queries the Database for the current time, and `/ping` returns `pong`
- **Postgres Database:** An empty PostgreSQL database with no tables or data. Used to show how to set up connectivity. The API applications execute `SELECT NOW() as now;` to determine the current time to return.

### Postgres

It's way more convenient to run postgres in a container as we saw in `04-using-3rd-party-containers`, so we will do that.

`make run-postgres` will start postgres in a container and publish port 5432 from the container to your localhost.

### api-node

To run the node api you will need to run `npm install` to install the dependencies (I used node `v19.4.0` and npm `v9.2.0`).

After installing the dependencies, `make run-api-node` will run the api in development mode with nodemon for restarting the app when you make source code changes.

### api-golang 

To run the golang api you will need to run `go mod download` to download and install the dependencies (I used `go1.19.1`)

After installing the dependencies, `make run-api-golang` will build and run the api.

### client-react

Like `api-node`, you will first need to install the dependencies with `npm install` (again, I used node `v19.4.0` and npm `v9.2.0`)

After installing the dependencies, `make run-client-react` will use vite to run the react app in development mode.

## 6. Building container image

### Building the Dockerfiles in this Repo

Each of the service subdirectories (./api-golang, ./api-node, ./client-react) contain a series of Dockerfiles (`Dockerfile.0` → `Dockerfile.N`) starting with the most simple naive approach, and improving them with each step.

The corresponding Makefiles also have a `build-N` target which can be used by:

```
cd api-golang && N=4 make build-N # This would build Dockerfile.4 of the api-golang component
```

Each image in the sequence should still function, with the final (highest #) being the one we will actually deploy later in the course.

---

### General Process

Dockerfiles generally have steps that are similar to those you would use to get your application running on a server.

1) Start with an Operating System
2) Install the language runtime
3) Install any application dependencies
4) Set up the execution environment
5) Run the application

***Note:** We can often jump right to #3 by choosing a base image that has the OS and language runtime preinstalled.*

### Writing Good Dockerfiles:

Here are some of the techniques demonstrated in the Dockerfiles within this repo:

1) **Pinning a specific base image:** By specifying an image tag, you can avoid nasty surprises where the base image
2) **Choosing a smaller base image:** There are often a variety of base images we can choose from. Choosing a smaller base image will usually reduce the size of your final image.
3) **Choosing a more secure base image:** Like image size, we should consider the number of vulnerabilities in our base images and the attack surface area. Chaingaurd publishes a number of hardened images (https://www.chainguard.dev/chainguard-images).
4) **Specifying a working directory:** Many languages have a convention for how/where applications should be installed. Adhering to that convention will make it easier for developers to work with the container.
5) **Consider layer cache to improve build times:** By undersanding the layered nature of container filesytems and choosing when to copy particular files we can make better use of the Docker caching system.
6) **Use COPY —link where appropriate:** The `--link` option was added to the `COPY` command in march 2022. It allows you to improve cache behavior in certain situations by copying files into an independent image layer not dependent on its predecessors.
7) **Use a non-root user within the container:** While containers can utilize a user namespace to differentiate between root inside the container and root on the host, this feature won't always be leveraged and by using a non-root user we improve the default safety of the container. When using Docker Desktop, the Virtual Machine it runs provides an isolation boundary between containers and the host, but if running Docker Engine it is useful to use a user namespace to ensure container isolation (more info here: https://docs.docker.com/engine/security/userns-remap/). This page also provides a good description for why to avoid running as root: https://cloud.google.com/architecture/best-practices-for-operating-containers#avoid_running_as_root.
8) **Specify the environment correctly:** Only install production dependencies for a production image, and specify any necessary environment variables to configure the language runtime accordingly.
9) **Avoid assumptions:** Using commands like `EXPOSE <PORT>` make it clear to users how the image is intended to be used and avoids the need for them to make assumptions.
10) **Use multi-stage builds where sensible:** For some situations, multi-stage builds can vastly reduce the size of the final image and improve build times. Learn about and use multi-stage builds where appropriate.

In general, these techniques impact some combination of (1) build speed, (2) image security, and (3) developer clarity. The following summarizes these impacts:

```
Legend:
 🔒 Security
 🏎️ Build Speed
 👁️ Clarity
```
- Pin specific versions [🔒 👁️]
  - Base images (either major+minor OR SHA256 hash) [🔒 👁️]
  - System Dependencies [🔒 👁️]
  - Application Dependencies [🔒 👁️]
- Use small + secure base images [🔒 🏎️]
- Protect the layer cache [🏎️ 👁️]
  - Order commands by frequency of change [🏎️]
  - COPY dependency requirements file → install deps → copy remaining source code [🏎️]
  - Use cache mounts [🏎️]
  - Use COPY --link [🏎️]
  - Combine steps that are always linked (use heredocs to improve tidiness) [🏎️ 👁️]
- Be explicit [🔒 👁️]
  - Set working directory with WORKDIR [👁️]
  - Indicate standard port with EXPOSE [👁️]
  - Set default environment variables with ENV [🔒 👁️]
- Avoid unnecessary files [🔒 🏎️ 👁️]
  - Use .dockerignore [🔒 🏎️ 👁️]
  - COPY specific files [🔒 🏎️ 👁️]
- Use non-root USER [🔒]
- Install only production dependencies [🔒 🏎️ 👁️]
- Avoid leaking sensitive information [🔒]
- Leverage multi-stage builds [🔒 🏎️]

### Additional Features

There are some additional features of Dockerfiles that are not shown in the example applications but are worth knowing about. These are highlighted in `Dockerfile.sample` and the corresponding build / run commands in the `Makefile`

1) **Parser directives:** Specify the particular Dockefile syntax being used or modify the escape character.
2) **ARG:** Enables setting variables at build time that do not persist in the final image (but can be seen in the image metadata).
3) **Heredocs syntax:** Enables multi-line commands within a Dockerfile.
4) **Mounting secrets:** Allows for providing sensitive credentials required at build time while keeping them out of the final image.
5) **ENTRYPOINT + CMD:** The interaction between `ENTRYPOINT` and `CMD` can be confusing. Depending on whether arguments are provided at runtime one or more will be used. See the examples by running `make run-sample-entrypoint-cmd`.
6) **buildx (multi-architecture images):** You can use a feature called `buildx` to create images for multiple architectures from a single Dockerfile. This video goes into depth on that topic: https://www.youtube.com/watch?v=hWSHtHasJUI. The `make  build-multiarch` make target demonstrates using this feature (and the images can be seen here: https://hub.docker.com/r/sidpalas/multi-arch-test/tags).

## 7. Container registries

A container registry is a repository, or collection of repositories, used to store and access container images. They serve as a place to store and share container images between developer systems, continuous integration servers, and deployment environments.

![](./readme-assets/container-registry.jpg)

Examples of popular container registries include:

- [Dockerhub](https://hub.docker.com)
- [Github Container Registry (ghcr.io.)](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)
- [Gitlab Container Registry](https://docs.gitlab.com/ee/user/packages/container_registry/)
- [Google Container Registry (gcr.io)](https://cloud.google.com/container-registry)
- [Amazon Elastic Container Registry (ECR)](https://aws.amazon.com/ecr/)
- [Azure Container Registry (ACR)](https://azure.microsoft.com/en-us/products/container-registry)
- [jFrog Container Registry](https://jfrog.com/container-registry/)
- [Nexus](https://blog.sonatype.com/nexus-as-a-container-registry)
- [Harbor](https://goharbor.io/)

## Authenticating to Container Registries

While you can pull many public images from registries without authenticating, in order to push images to a registry, or pull a private image, you will need to authentic

Docker can login directly to some registries with basic authentication (username/password) or call out to separate programs known as credential helpers. For example, to authenticate to the Google Container Registry, docker uses the `gcloud` command line utility from GCP (https://cloud.google.com/container-registry/docs/advanced-authentication#gcloud-helper). 

If available, Docker can also store the credentials in a secure store (`macOS keychain`, `Windows Credential Manager`) to help protect those credentials.

![](./readme-assets/credential-helper.jpg)