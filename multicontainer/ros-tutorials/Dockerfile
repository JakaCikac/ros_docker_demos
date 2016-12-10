# This is a Dockerfile for ros:ros-tutorials
FROM ros
MAINTAINER ruffsl roxfoxpox@gmail.com

# install ros tutorials packages
RUN apt-get update && apt-get install -y \
    ros-kinetic-ros-tutorials \
    ros-kinetic-common-tutorials \
    && rm -rf /var/lib/apt/lists/*
