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