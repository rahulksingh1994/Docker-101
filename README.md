# docker101

Docker Introduction: <https://docs.microsoft.com/en-us/dotnet/architecture/microservices/container-docker-introduction>

## Overview of Docker Commands

### Images

```bash
# Display local images
docker images 
# Pull an image from a repository
docker pull  
# Tag image with a nickname that will reference the source image
docker tag 
# Save container changes to an image
docker commit 
# Push an image up to a repository
docker push 
# Build an image from the Dockerfile
docker build
```

### Containers

```bash
# Display current containers
docker ps
# Run a container from an image
docker run
# Start a new process to run commands in a running container
docker exec
# Display container logs
docker logs
#  Stop a running container
docker kill
# Delete a container
docker rm
```

### Networks

```bash
# Display current networks
docker network ls
# Create a network
docker network create
# Connect containers to a network
docker network connect
# Display network details including connected containers
docker network inspect
```

## Getting a Docker Image

```bash
# See local images
docker images

# Pull an image of the ngsa-app from the GitHub registry
docker pull ghcr.io/retaildevcrews/ngsa-app:beta

# See the pulled image locally
docker images

# Tag the image to used a shortened name (ngsa-app:beta)
docker tag ghcr.io/retaildevcrews/ngsa-app:beta ngsa-app:beta

# View the tagged image which references a source image
docker images
```

## Running a Container

```bash
# See created containers
# -a - Show all containers (default shows just running)
docker ps -a

# Start a container with a running image
# --name - Naming the container
# -d - Detached
# --in-memory - Not a docker option. Passed in flag to the ngsa-app via Dockerfile entrypoint
docker run --name ngsa -d ngsa-app:beta --in-memory

# See running containers
docker ps

# See running containers
# --no-trunc - Don't truncate output to be able to see COMMAND
docker ps --no-trunc

# Run a command in the running container
# -i - Interactive. Keep STDIN open even if not attached
# -t - Allocate a pseudo-TTY
docker exec -it ngsa sh

# The prompt changed
# We are now "in" the docker container

# Run some commands in the container
ls -al

# Leave the container
exit

# Fetch the logs of the container
docker logs ngsa
```

## Ports

```bash
# Remove the container because we can't expose ports on a running container
docker rm ngsa

# Force the removal of a running container
# -f - Force the removal of a running container
docker rm -f ngsa

# Verify the container was removed
docker ps -a

# Start a container with an exposed port
# --name - Naming the container
# -d - Detached
# --rm - Automatically remove container when stopped
# -p - Publish a container's port(s) to the host (HOST_PORT:CONTAINER_PORT)
# --in-memory - Not a docker option. Passed in flag to ngsa-app via image's ENTRYPOINT
docker run --name ngsa -d --rm -p 80:8080 ngsa-app:beta --in-memory

# Verify app on exposed port
http localhost:80/version
```

## Network

```bash
# Start a container
# --name - Naming the container
# -d - Detached
# --rm - Automatically remove container when stopped
# --entrypoint - Overwrites the image's default ENTRYPOINT which states the start of a command and tacks on the rest from docker run
docker run --name webvalidate -d --rm --entrypoint sh ghcr.io/asb-spark/docker101-webv:spark -c "sleep 999999d"

# Verify the container is running
docker ps 

# Show networks
docker network ls

# Create a network
docker network create web

# Connect containers to the network
docker network connect web ngsa
docker network connect web webvalidate

# Verify containers were added to network
docker network inspect web

# Execute WebV load test on ngsa-app via network call
# -s - Not a docker option. Passed in flag to WebV via image's ENTRYPOINT.
#      ngsa is the container on the network.
#      8080 is the port ngsa-app is listening on in the ngsa container.
# -f - Not a docker option. Passed in flag to WebV via image's ENTRYPOINT
docker exec webvalidate dotnet ../aspnetapp.dll -s http://ngsa:8080 -f benchmark.json

# Output should show load test "Failed: * Errors"
# benchmark.json file on the webvalidate container needs to be updated
```

## Volumes

```bash
# Kill container
docker kill webvalidate

# Verify container was removed when stopped
docker ps -a

# Run the container and mount the file we want to edit
# --name - Naming the container
# -d - Detached
# --rm - Automatically remove container when stopped
# --entrypoint - Overwrites the image's default ENTRYPOINT which states the start of a command and tacks on the rest from docker run
# --net - Connect a container to a network
# -v - Bind mount a volume
docker run --name webvalidate -d --rm --entrypoint sh --net web -v $(pwd)/webvalidate/src/app/benchmark.json:/app/TestFiles/benchmark.json ghcr.io/asb-spark/docker101-webv:spark -c "sleep 999999d"

# Update the app/TestFiles/benchmark.json in webvalidate container
# Replace 'zzz' with 'api'

# Execute WebV load test on ngsa-app with updated benchmark.json
docker exec webvalidate dotnet ../aspnetapp.dll -s http://ngsa:8080 -f benchmark.json

# Output should no longer show "Failed: * Errors"
```

## Commit Image Layer

```bash
# Try running new container with image to see fixed load test
# --name - Naming the container
# -d - Detached
# --rm - Automatically remove container when stopped
# --entrypoint - Overwrites the image's default ENTRYPOINT which states the start of a command and tacks on the rest from docker run
# --net - Connect a container to a network
docker run --name webvalidate-fix -d --rm --entrypoint sh --net web ghcr.io/asb-spark/docker101-webv:spark -c "sleep 99999d"

# Try to execute fixed load test
docker exec webvalidate-fix dotnet ../aspnetapp.dll -s http://ngsa:8080 -f benchmark.json

# benchmark.json only changed in container
# benchmark.json wasn't updated in image
# New containers from that image won't have the fix

# **NOTE**
# Can commit changes in a container to an image
# The commit operation will not include any data contained in volumes mounted inside the container
# In the webvalidate container, the benchmark.json was mounted, and its changes cannot be commited
# Will make change directly in webvalidate-fix container to commit change

# Open shell in container
# -i - Interactive. Keep STDIN open even if not attached
# -t - Allocate a pseudo-TTY
docker exec -it webvalidate-fix sh

# Open file in container
vi benchmark.json

# Edit file in container
:%s/zzz/api/g

# Save file in container
:wq

# Leave the container
exit

# Verify successful load test
docker exec webvalidate-fix dotnet ../aspnetapp.dll -s http://ngsa:8080 -f benchmark.json

# Commit container change to image
docker commit webvalidate-fix docker101-webv:fix

# Verify new image
docker images

# Kill container to re-use name
docker kill webvalidate-fix

# Run newly create image in container
# --name - Naming the container
# -d - Detached
# --rm - Automatically remove container when stopped
# --entrypoint - Overwrites the image's default ENTRYPOINT which states the start of a command and tacks on the rest from docker run
# --net - Connect a container to a network
docker run --name webvalidate-fix -d --rm --entrypoint sh --net web docker101-webv:fix -c "sleep 99999d"

# Verify fixed load test
docker exec webvalidate-fix dotnet ../aspnetapp.dll -s http://ngsa:8080 -f benchmark.json
```

## Dockerfile

```bash
# Make sure ./webvalidate/src/app/benchmark.json in Codespaces is update by replacing 'zzz' with 'api'

# Build image from Dockerfile
docker build ./webvalidate -t docker101-webv:dockerfile

# Verify new image
docker images

# Run newly create image in container
# -d - Detached
# --name - Naming the container
# --rm - Automatically remove container when stopped
# --entrypoint - Overwrites the image's default ENTRYPOINT which states the start of a command and tacks on the rest from docker run
# --net - Connect a container to a network
docker run -d --name webvalidate-dockerfile --rm --entrypoint sh --net web docker101-webv:dockerfile -c "sleep 99999d"

# Verify fixed load test
docker exec webvalidate-dockerfile dotnet ../aspnetapp.dll -s http://ngsa:8080 -f benchmark.json
```

## Notes

Volumes - virtual "discs" to store and share data

* persistent - shared between host and container
  * docker run -ti -v /Users/root/example:/shared-folder ubuntu bash
* ephemeral - sticks around while being used by container
  * docker run -ti -v /shared-data --name containerA ubuntu bash
  * docker run -ti --volumes-from containerA --name containerB ubuntu bash

docker system prune -a / docker system prune -f

* It often gets GBs of disk space back. Don't use if debugging big docker build as it will delete all of your cached build layers and you'll rebuild from scratch
* Remove all unused containers, networks, images (both dangling and unreferenced), and optionally, volumes.

Dockerfile Commands

* FROM - which image to download and start from
  * must be first command in dockerfile
  * multiple from commands means dockerfile produces more than one image
* MAINTAINER
* RUN - runs the command line, waits for finish, and saves result
* ADD - adds local files, add the contents of tar archives, and downloads from URLS
* ENV - sets environment variables and available during build in in resulting image
* ENTRYPOINT - start of command and tacks on rest of command from docker run
* CMD - specifies whole command
  * docker run command replaces CMD
* ENTRYPOINT + CMD - runs one after another
* Command syntax for ENTRYPOINT, RUN, and CMD
  * shell form: nano notes.txt
    * called surrounded by a shell, like bash
  * exec form: ["/bin/nano", "notes.txt"]
    * calls directly
    * slightly more efficient
* EXPOSE - exposes port to container
* VOLUME
  * shared volumes: VOLUME ["/host/path/" "/container/path/"]
  * ephemeral volumes: VOLUME ["/shared-data"]
* WORKDIR - sets directory for remainder of Dockerfile and resulting container when run
* USER - sets the user the container will run as
