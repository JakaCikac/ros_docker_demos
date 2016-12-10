# Multi-Container ROS nodes
In this tutorial we'll use Docker to deploy ROS nodes onto a separate multiple containers on a single host and connect them together on a virtual network.

## Dependencies
Here is a breakdown of the dependencies across our demo application:

#### Local
* [Docker](https://www.docker.com/)
* [Docker Compose](https://docs.docker.com/compose/)

#### Image
* [ROS](https://registry.hub.docker.com/_/ros/)

## Setup

### Local
For our local setup, we'll need to install Docker along with the other Docker tools so that we can create and run multiple containers to deploy our setup. Fallow the links above for installation guides for you platform.

## Image
For our image setup, we'll need to build an image from the Docker Hub's official ROS repo to include the necessary demo packages we'll be using.
> From within this demo directory, we can build our needed demo images:

    docker build --tag ros:ros-tutorials ros-tutorials/.

## Deploy

### Creating a network

If we want our all ROS nodes to easily talk to each other, we'll can use a virtual network to connect the separate containers. In this short example, we'll create a virtual network, spin up a new container running `roscore` advertised as the `master` service on the new network, then spawn a message publisher and subscriber process as services on the same network.

### Build image

> Build a ROS image that includes ROS tutorials using this `Dockerfile:`

```dockerfile
FROM ros
# install ros tutorials packages
RUN apt-get update && apt-get install -y
    ros-kinetic-ros-tutorials \
    ros-kinetic-common-tutorials \
    && rm -rf /var/lib/apt/lists/
```

> Then to build the image from within the same directory:

```console
$ docker build --tag ros:ros-tutorials .
```

#### Create network

> To create a new network `foo`, we use the network command:

	docker network create foo

> Now that we have a network, we can create services. Services advertise there location on the network, making it easy to resolve the location/address of the service specific container. We'll use this make sure our ROS nodes can find and connect to our ROS `master`.

#### Run services

> To create a container for the ROS master and advertise it's service:

```console
$ docker run -it --rm \
    --net foo \
    --name master \
    ros:ros-tutorials \
    roscore
```

> Now you can see that master is running and is ready manage our other ROS nodes. To add our `talker` node, we'll need to point the relevant environment variable to the master service:

```console
$ docker run -it --rm \
    --net foo \
    --name talker \
    --env ROS_HOSTNAME=talker \
    --env ROS_MASTER_URI=http://master:11311 \
    ros:ros-tutorials \
    rosrun roscpp_tutorials talker
```

> Then in another terminal, run the `listener` node similarly:

```console
$ docker run -it --rm \
    --net foo \
    --name listener \
    --env ROS_HOSTNAME=listener \
    --env ROS_MASTER_URI=http://master:11311 \
    ros:ros-tutorials \
    rosrun roscpp_tutorials listener
```

> Alright! You should see `listener` is now echoing each message the `talker` broadcasting. You can then list the containers and see something like this:

```console
$ docker service ls
SERVICE ID          NAME                NETWORK             CONTAINER
67ce73355e67        listener            foo                 a62019123321
917ee622d295        master              foo                 f6ab9155fdbe
7f5a4748fb8d        talker              foo                 e0da2ee7570a
```

> And for the services:

```console
$ docker ps
CONTAINER ID        IMAGE               COMMAND                CREATED              STATUS              PORTS               NAMES
a62019123321        ros:ros-tutorials   "/ros_entrypoint.sh    About a minute ago   Up About a minute   11311/tcp           listener
e0da2ee7570a        ros:ros-tutorials   "/ros_entrypoint.sh    About a minute ago   Up About a minute   11311/tcp           talker
f6ab9155fdbe        ros:ros-tutorials   "/ros_entrypoint.sh    About a minute ago   Up About a minute   11311/tcp           master
```

#### Introspection

> Ok, now that we see the two nodes are communicating, let get inside one of the containers and do some introspection what exactly the topics are:

```console
$ docker exec -it master bash
$ source /ros_entrypoint.sh
```

> If we then use `rostopic` to list published message topics, we should see something like this:

```console
$ rostopic list
/chatter
/rosout
/rosout_agg
```

#### Tear down

> To tear down the structure we've made, we just need to stop the containers and the services. We can stop and remove the containers using `Ctrl^C` where we launched the containers or using the stop command with the names we gave them:

```console
$ docker stop master talker listener
$ docker rm master talker listener
```

### Compose

Now that you have an appreciation for bootstrapping a distributed ROS example manually, lets try and automate it using [`docker-compose`](https://docs.docker.com/compose/)\.

> Start by making a folder named `rostutorials` and moving the Dockerfile we used earlier inside this directory. Then create a yaml file named `docker-compose.yml` in the same directory and paste the following inside:

```yaml
version: '2'
services:
  master:
    build: .
    container_name: master
    command:
      - roscore

  talker:
    build: .
    container_name: talker
    environment:
      - "ROS_HOSTNAME=talker"
      - "ROS_MASTER_URI=http://master:11311"
    command: rosrun roscpp_tutorials talker

  listener:
    build: .
    container_name: listener
    environment:
      - "ROS_HOSTNAME=listener"
      - "ROS_MASTER_URI=http://master:11311"
    command: rosrun roscpp_tutorials listener
```

> Now from inside the same folder, use docker-compose to launch our ROS nodes and specify that they coexist on their own network:

```console
$ docker-compose up -d
```

> Notice that a new network named `rostutorials` has now been created. We can monitor the logged output of each service, such as the listener node like so:

```console
$ docker-compose logs listener
```

> Finally, we can stop and remove all the relevant containers using docker-copose from the same directory:

```console
$ docker-compose stop
$ docker-compose down
```

> Note: the auto-generated network, `rostutorials`, will persist over the life of the docker engine or until you explicitly remove it using [`docker network rm`](https://docs.docker.com/engine/reference/commandline/network_rm/)\.
