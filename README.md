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