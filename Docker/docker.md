Docker is used to bundle an application and all its required dependencies into a single lightweight package, called a container, so that the software runs consistently across any computer environment.



\-->  It effectively eliminates the common "it works on my machine" problem by ensuring that development, testing, and production setups are identical.



\--> Docker wraps your application and everything it needs to run into a neat, standardized package called a container.



\--> windows user developed an application. Now, new mac-user comes and want to do changes. so he want to replicate the entire application into his own system so that he can also work on the same application.

&#x20;

The mac user installs the dependencies manually one by one which can lead to manual errors like:

i) Every real life application has a lot of dependencies.(which takes a lot time install manually)

ii) the specific version of the dependencies should be installed

&#x20;    ex: developed application: node: v16, expres: 3.12.11 ,....

iii) command-line-interface commands ---> differ from windows , mac ...

\--> these problems could encountered not only in local environment but also in production level environment.



"IT works on my machine" --> common problem that encounters.



\--> Docker helps us to build Containers \& Images.



To understand Docker, you only need to know three basic terms:



Docker File: A simple text file containing a list of commands/instructions on how to build your application's environment (e.g., "Install Ubuntu, install Python, copy my code, run port 8080").



Image: A read-only snapshot of your application built from the Docker file. It contains your code, libraries, and dependencies. Think of it as the blueprint.



Container: A live, running instance of an image. This is the actual "shipping container" executing your app.



\--> Docker is used across the entire lifecycle of software development and deployment. Docker is used for

local development, CI/CD, Microservices Architecture, cloud deployment.



# **What Problems Does Docker Solve?**



1\. The "It Works on My Machine" Syndrome

The Problem: A developer writes code that works perfectly on their Mac. They hand it off to a tester using Windows, or deploy it to a Linux server, and it breaks because of a slight difference in operating systems, software versions, or configurations.



The Solution: Because a Docker container packages the application and its exact environment together, it will run identical to how it did on your machine, whether it's running on a teammate's laptop, a testing server, or the cloud.



2\. Dependency Hell

The Problem: Project A requires Python 2.7, but Project B requires Python 3.11. Trying to install and manage different versions of the same software on one computer often corrupts paths and breaks things.



The Solution: Docker completely isolates applications. Project A lives in its own container with Python 2.7, and Project B lives in a separate container with Python 3.11. They never interact or conflict.



# **Container:**



\--> Docker Container: It's a single bundle/ unit contains application along with all dependencies needed to run that application.

&#x20;        Container = single-unit of \[application + dependencies]. i.e. replicating the application over different machines with different OS becomes easy.



A Docker container is a lightweight, standalone, and executable package of software that includes everything needed to run an application. It bundles your application code, runtime, system tools, libraries, and configurations into a single standardized unit. This ensuring that the application runs quickly, safely, and consistently across any computing environment, eliminating the classic "it works on my machine" problem.



A container is a standard unit of software that packages up code and all its dependencies so the application runs quickly and reliably from one computing environment to another.



#### Key Benefits of container:

\--> **i) Portability:** Write your application once and run it anywhere—on a local laptop, cloud platforms like AWS Elastic Container Service, or an on-premise server.

"Write Once, Run Anywhere": Because a container packages your application along with its exact runtime, libraries, and configuration settings, it behaves identically everywhere.



\--> Eliminates Environment Drift: No more tracking down why code runs perfectly in your development environment but mysteriously crashes in testing or production. The exact same container image moves down the pipeline.



\--> **ii)** **Light Weight:** It is **light weight** in nature.containers do not bundle a full operating system. Instead, they share the host computer’s operating system kernel and only include the bare minimum software (like Python, Node.js, or specific libraries) needed to run the app.



\--> **iii) Isolation:** Each container runs as an isolated process. If one container crashes or gets compromised, it won't affect the rest of your system.



\--> **iv) Scalability:** Because they are so lightweight, you can spin up thousands of containers across a cluster using orchestration systems like Kubernetes.



\--> **v) Immutable Infrastructure:** Since container behavior is locked into the image, your testing environment will exactly match your production environment.



**The Lifecycle of a Docker Container:**

**\[ Docker Image ] ---> ( docker run ) ---> \[ Running Container ] ---> ( docker stop ) ---> \[ Stopped Container ]**



**Basic Docker Commands:**



**>** **docker run** (Born): Docker takes an image, creates a fresh writeable layer on top of it, and starts your application. It goes from non-existent to running in less than a second.



**>** **docker stop** **(Asleep):** The application inside is paused or halted. The container still exists on your disk, and any data created while it was running is preserved, but it isn't using any CPU or RAM.



**> docker start (Awake):** Wakes up a stopped container right where it left off.



> \*\*docker rm (Destroyed)\*\*: Permanently deletes the container.



* docker pull <image>: Downloads a premade image from an online registry like Docker Hub.
* docker run <image>: Creates and starts a container from a specified image.
* docker ps: Lists all currently running containers on your system.
* docker stop <container\_id>: Gracefully halts a running container.
* docker rm <container\_id>: Permanently deletes a stopped container from memory.



# **Images:**



**A Docker image is a lightweight, standalone, and read-only file that serves as a blueprint for creating a running container. It contains everything your software needs to run, including the application code, runtime environments, system tools, libraries, and configurations.** 



\[ Dockerfile ]   ---> Build --->   \[ Docker Image ]   ---> Run --->   \[ Docker Container ]



**A Docker image is an unchangeable (read-only) file**.



You cannot "run" an image directly; instead, you use an image as a template to stamp out one or more live containers.

You don't always have to build images from scratch. Docker images are stored and shared in central repositories called **Registries.**



Class (Blueprint to build multiple objects)  ---> Object 

Docker Image (blue print to build multiple containers)---> build Docker Containers



**Note: We always share Docker Image but not container.**

**--> we download the image and create container in our local system.**

**--> inside container we have running environment set-up.**



**class**(No-memory) --> object (uses memory)

Docker Image is an inert, lightweight template, and a Docker Container is the running, instantiated execution of that image.





**Docker container is an actual running instance where as Docker Image is a static snapshot of what the local development should look like.**



> docker run -it ubuntu (we can run ubuntu OS on our local machine within that specific container).

\-- When you type docker run -it ubuntu into your terminal, you are telling Docker to download a bare-bones Ubuntu Linux operating system, spin it up inside an isolated container, and immediately drop you into its command line.



* docker run: This tells the Docker Engine to create a brand-new container and start it.



* \-i (Interactive): Keeps the container's standard input (stdin) open. Even if you aren't attached to the container, it stays open and listens for you to type commands.



* \-t (TTY): Allocates a pseudo-TTY (a virtual terminal screen). This bridges your computer's terminal to the container's terminal, giving you the familiar prompt (like root@f3a2b1c4e5d6:/#) and allowing you to see colors, use autocomplete with Tab, and use keyboard shortcuts.
 


* ubuntu: This specifies the Docker Image to use. Docker will look for an official Ubuntu Linux image on your machine. If you don't have it locally, it will automatically download (pull) it from Docker Hub first.





## **Docker Hub:** 

public collection of docker images (https://hub.docker.com/)



> difference between   machine and container?

The primary difference between Docker and a Virtual Machine (VM) is that Docker virtualizes the operating system layer by sharing the host machine's kernel, while a Virtual Machine virtualizes the underlying physical hardware to run a complete, independent guest operating system.



**Docker Commands:** 

> docker pull <Image\_name>

> docker images

> docker run <Image\_name>

> docker run -it <Image\_name>

> docker stop CONTAINER\_NAME or CONTAINER\_ID --> Safely stops a running container.

> docker start CONTAINER\_NAME or CONTAINER\_ID --> Wakes up a stopped container.

> docker restart <container\_id> --> docker restart <container\_id>

> docker rm <container\_id> --> Deletes a stopped container permanently.

EX: 

> docker pull hello-world

> docker run hello-world

# **Docker Commands:**

> docker images --> (Lists all the Docker images currently saved on your computer's disk.)

❯ docker ps --> (Lists all currently running containers.)

❯ docker ps -a --> (Lists all containers (running, stopped, or exited).)

> docker run <Image\_Name> --> Creates and starts a brand-new container from an image.

> docker pull <Image\_Name> --> Downloads an image from an online registry (like Docker Hub) without running it.

> docker run -it <Image\\\_name> --> (Interactive mode) (allows to access container terminal which inputs and outputs anything). 

&#x20;   > docker pull ubuntu	

&#x20;   > docker run -it ubuntu

&#x20;   > ls (list all files \& folders inside that container)

&#x20;   > env command used to view environment variables inside a running container.

&#x20;   > exit

> docker stop CONTAINER\_NAME or CONTAINER\_ID --> Safely stops a running container.

> docker start CONTAINER\_NAME or CONTAINER\_ID --> Wakes up a stopped container.

> docker restart <container\_id> --> docker restart <container\\\_id>

> docker rm <container\_id> --> Deletes a stopped container permanently.

> docker rmi <image\_id or image\_name> --> Deletes a Docker image from your computer to free up space.

**Note: To delete docker image , first we need to delete docker containers of that image and then delete image.**

> docker pull <Image\_Name>:version

> docker run -d <Image\_Name>

> docker run --name <CONT\_Name> -d <Image\_Name>





Tags are basically the versions of our images.

Docker Image --> Name, Tag(version), ImageID, Created, size, actions. 

Docker Conatainer --> Random\_Name of container(By default), ContainerID, Image, PORT, cpu(%), last started , actions.

Docker Tags --> A tag is like a version or a variant of the same docker image.

>  docker pull mysql --> By default pulls latest version.

>  docker pull mysql:8.0 --> to pull differnt tags 

> docker run -d -e MYSQL\_ROOT\_PASSWORD=secret --name mysql--older mysql:8.0 (--name is to give custom name)



### Docker Image Layers:

> An Image has a differnt layers 



**Every docker image is made up of different layers(collection of layers).These layers are immutable(cannot change) and read-only.**



Base Layer --> Layer1 --> layer2 .....--> container 



### Port Binding: 

> docker run -p8080:3306 <Image\_Name>

\--> -p ==> indicates host port

\[host port] ------(map)----> \[container port]



\--> By default docker containers have a port which is bounded to them.

Port binding in Docker maps a specific port on your host machine to a port inside your container, making containerized services accessible to the outside world.

**> docker run -d -e MYSQL\_ROOT\_PASSWORD=secret --name mysql-latest -p8080:3306 mysql**

**Note: Host machine ports \& Container ports are different from one another.**



### Troubleshoot Commands:

> docker logs CONT\_ID ==> Shows the console output (stdout) of what is happening inside the container.

> docker exec -it CONT\_ID /bin/bash (Runs a brand-new command inside an already running container. Often used to open a terminal.)

\--> to run additional commands on already running container.

> docker exec -it CONT\_ID /bin/sh



\--> Docker desktop adds a light-weight hypervisor layer to OS i.e. which uses Linux where the containers run.



## Docker Networks:

> docker network ls

> docker network create <NETWORK\_NAME>



> docker run -d \\

&#x20; -p27017:27017 \\

&#x20; --name <mongo> \\

&#x20; --network <mongo-network>\\

&#x20; -e User=admin (environment variables) \\

&#x20; -e ...

&#x20; <Image\_name>

## Docker Compose:

docker compose is a tool for defining and running multi-container applications.



we use yaml file to store docker command, that file is called a Docker Compose YAML file (compose.yaml or docker-compose.yml), which translates long, messy terminal commands into a clean, reusable blueprint.



> docker compose -f fileName.yaml up -d

> docker compose -f fileName.yaml down





### Dockerizing our App:

docker file instructions:

FROM, WORKDIR, COPY, RUN, CMD, EXPOSE, ENV



### Publishing images:

> docker build -t <>

> docker login

> docker logout

> docker push <>



### Docker Volumes:

volumes are persistent data stores for containers.

> docker volume ls

> docker volume create <VOL\_Name>

> docker volume rm <VOL\_Name>

\--> Volumes are isolated and not attach with any running container.



\--> to attach volumes to running containers

> docker run -v VOL\_NAME:CONT\_DIR
> docker run -v MOUNT\_PATH

> docker run -v HOST\_DIR:CONT\_DIR





------------------------------------------------------------------------

# 1. Base image with fixed version
FROM python:3.12-slim

# 2. Build-time argument for custom setup
ARG APP_ENV=production

# 3. Persistent environment variables
ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    PORT=8000 \
    ENVIRONMENT=${APP_ENV}

# 4. Set the container working directory
WORKDIR /app

# 5. Leverage Docker layer caching for dependencies
COPY requirements.txt .

# 6. Build instruction: Install dependencies
RUN pip install --no-cache-dir -r requirements.txt

# 7. Copy the rest of the application files
COPY . .

# 8. Security: Run container as non-root user
RUN useradd -u 8888 appuser && chown -R appuser:appuser /app
USER appuser

# 9. Document the runtime port
EXPOSE 8000

# 10. Default runtime command
CMD ["python", "app.py"]
-------------------------------------------------------------------------









