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
Run the Whalesay Container with the following command:
``` docker run docker/whalesay cowsay "Hey Team! 👋" ``` 



