## Container Fundamentals & Filesystem Architecture

### A. Container Layer Structure
- **Read-only image layer**: When a container is created from an image, the image serves as an immutable foundation
- **Read/write layer overlay**: Docker automatically adds a thin read/write layer on top for runtime modifications
- **Isolation principle**: Each container gets its own unique read/write layer, completely isolated from other containers
- **Container lifecycle**: When a container is removed (especially with `--rm` flag), its read/write layer is permanently destroyed

### B. Demonstrating Container Ephemerality

```bash
# First container session - installing ping utility
docker run --interactive --tty --rm ubuntu:22.04
apt update
apt install iputils-ping --yes
ping google.com -c 1  # Succeeds
exit

# Second container session - ping is gone
docker run --interactive --tty --rm ubuntu:22.04
ping google.com -c 1  # Fails with "command not found"
```

### C. Container Persistence (Without Volumes)

```bash
# Named container without auto-removal
docker run -it --name my-ubuntu-container ubuntu:22.04
apt update && apt install iputils-ping --yes
exit

# Container inspection & restart
docker container ps -a | grep my-ubuntu-container
docker container inspect my-ubuntu-container
docker start my-ubuntu-container
docker attach my-ubuntu-container
ping google.com -c 1  # Now succeeds in this specific container
```

## Proper Data Persistence Approaches

### A. Building Dependencies into Images (Dockerfile)

```bash
# Creating custom image with dependencies pre-installed
docker build --tag my-ubuntu-image -<<EOF
FROM ubuntu:22.04
RUN apt update && apt install iputils-ping --yes
EOF

# Running container from custom image
docker run -it --rm my-ubuntu-image
ping google.com -c 1  # Pre-installed, works immediately
```

### B. Volume Mounts for Application Data

#### i. Creating & Using Docker Volumes

```bash
# Create a named volume
docker volume create my-volume

# Create container with mounted volume
docker run -it --rm --mount source=my-volume,destination=/my-data/ ubuntu:22.04
# Alternative shorter syntax
docker run -it --rm -v my-volume:/my-data ubuntu:22.04

# Create data in the volume
echo "Hello from the container!" > /my-data/hello.txt
exit

# Verify data persistence in a new container
docker run -it --rm -v my-volume:/my-data ubuntu:22.04
cat my-data/hello.txt  # File still exists
```

#### ii. Volume Storage Location
- On Linux: `/var/lib/docker/volumes/volume-name/_data`
- On Docker Desktop: Inside the Linux VM managed by Docker Desktop
- Can be accessed using specialized container:
  ```bash
  # Accessing Docker host filesystem
  docker run -it --rm --privileged --pid=host justincormack/nsenter1@sha256:5af0be5e42ebd55eea2c593e4622f810065c3f45bb805eaacf43f08f3d06ffd8
  ls /var/lib/docker/volumes/my-volume/_data
  ```

### C. Bind Mounts for Configuration Files

```bash
# Mount host directory into container
docker run -it --rm --mount type=bind,source="${PWD}"/my-data,destination=/my-data ubuntu:22.04
# Shorter syntax
docker run -it --rm -v ${PWD}/my-data:/my-data ubuntu:22.04

# Create data that will be visible on host
echo "Hello from the container!" > /my-data/hello.txt
exit

# View file directly on host
cat my-data/hello.txt
```

## Production Database Containers

### A. Postgres Configuration

```bash
# Basic setup with volume and password
docker run -d --rm \
  -v pgdata:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=foobarbaz \
  -p 5432:5432 \
  postgres:15.1-alpine

# Advanced setup with custom configuration
docker run -d --rm \
  -v pgdata:/var/lib/postgresql/data \
  -v ${PWD}/postgres.conf:/etc/postgresql/postgresql.conf \
  -e POSTGRES_PASSWORD=foobarbaz \
  -p 5432:5432 \
  postgres:15.1-alpine -c 'config_file=/etc/postgresql/postgresql.conf'
```

### B. MongoDB Configuration

```bash
# Basic setup
docker run -d --rm \
  -v mongodata:/data/db \
  -e MONGO_INITDB_ROOT_USERNAME=root \
  -e MONGO_INITDB_ROOT_PASSWORD=foobarbaz \
  -p 27017:27017 \
  mongo:6.0.4

# With custom configuration
docker run -d --rm \
  -v mongodata:/data/db \
  -v ${PWD}/mongod.conf:/etc/mongod.conf \
  -e MONGO_INITDB_ROOT_USERNAME=root \
  -e MONGO_INITDB_ROOT_PASSWORD=foobarbaz \
  -p 27017:27017 \
  mongo:6.0.4 --config /etc/mongod.conf
```

### C. Redis Configuration

```bash
# Simple setup (with optional persistence)
docker run -d --rm \
  -v redisdata:/data \
  redis:7.0.8-alpine

# With custom configuration
docker run -d --rm \
  -v redisdata:/data \
  -v ${PWD}/redis.conf:/usr/local/etc/redis/redis.conf \
  redis:7.0.8-alpine redis-server /usr/local/etc/redis/redis.conf
```

### D. MySQL Configuration

```bash
# Basic setup
docker run -d --rm \
  -v mysqldata:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=foobarbaz \
  -p 3306:3306 \
  mysql:8.0.32

# With custom configuration directory
docker run -d --rm \
  -v mysqldata:/var/lib/mysql \
  -v ${PWD}/conf.d:/etc/mysql/conf.d \
  -e MYSQL_ROOT_PASSWORD=foobarbaz \
  -p 3306:3306 \
  mysql:8.0.32
```

### E. Elasticsearch Configuration

```bash
docker run -d --rm \
  -v elasticsearchdata:/usr/share/elasticsearch/data \
  -e ELASTIC_PASSWORD=foobarbaz \
  -e "discovery.type=single-node" \
  -p 9200:9200 \
  -p 9300:9300 \
  elasticsearch:8.6.0
```

### F. Neo4j Configuration

```bash
docker run -d --rm \
  -v=neo4jdata:/data \
  -e NEO4J_AUTH=neo4j/foobarbaz \
  -p 7474:7474 \
  -p 7687:7687 \
  neo4j:5.4.0-community
```

## Interactive Development & Testing Environments

### A. Operating System Environments

```bash
# Ubuntu
docker run -it --rm ubuntu:22.04

# Debian
docker run -it --rm debian:bullseye-slim

# Alpine (lightweight)
docker run -it --rm alpine:3.17.1

# BusyBox (minimal utilities)
docker run -it --rm busybox:1.36.0
```

### B. Programming Runtime Environments

```bash
# Python
docker run -it --rm python:3.11.1

# Node.js
docker run -it --rm node:18.13.0

# PHP
docker run -it --rm php:8.1

# Ruby
docker run -it --rm ruby:alpine3.17
```

## V. CLI Utility Containers

### A. Text & Data Processing Tools

```bash
# jq (JSON processor)
docker run -i stedolan/jq <sample-data/test.json '.key_1 + .key_2'

# yq (YAML processor)
docker run -i mikefarah/yq <sample-data/test.yaml '.key_1 + .key_2'

# GNU sed
docker run -i --rm busybox:1.36.0 sed 's/file./file!/g' <sample-data/test.txt

# GNU base64
echo "This string is just long enough to trigger a line break in GNU base64." | \
  docker run -i --rm busybox:1.36.0 base64
```

### B. Cloud CLI Tools

```bash
# AWS CLI with credential mounting
docker run --rm -v ~/.aws:/root/.aws amazon/aws-cli:2.9.18 s3 ls

# Google Cloud CLI
docker run --rm -v ~/.config/gcloud:/root/.config/gcloud \
  gcr.io/google.com/cloudsdktool/google-cloud-cli:415.0.0 gsutil ls
```

### C. Improving CLI Ergonomics

```bash
# Shell function approach
yq-shell-function() {
  docker run --rm -i -v ${PWD}:/workdir mikefarah/yq "$@"
}
yq-shell-function <sample-data/test.yaml '.key_1 + .key_2'

# Alias approach
alias 'yq-alias=docker run --rm -i -v ${PWD}:/workdir mikefarah/yq'
yq-alias <sample-data/test.yaml '.key_1 + .key_2'
```

## Docker Architecture Requirements

### A. Linux Kernel Features Required
- **Union mount file systems**: Allows files and directories from separate filesystems to merge into a single cohesive directory structure in a new virtual filesystem
- **Control Groups (cgroups)**: Linux kernel feature for organizing processes and allocating resources
- **Namespaces**: Isolation mechanism that wraps global system resources in isolated instances; changes to global resources are only visible to processes within the same namespace

### B. Docker Desktop Architecture
- On Windows/Mac, Docker runs inside a lightweight Linux VM
- Container processes execute within this VM
- Filesystem operations occur within VM with optional mappings to host
- Network port forwarding connects container ports to host ports

## Docker Command Reference

### A. Container Lifecycle
- `docker run`: Create and start a container
  - `-i, --interactive`: Keep STDIN open
  - `-t, --tty`: Allocate a pseudo-TTY
  - `--rm`: Automatically remove container when it exits
  - `--name NAME`: Assign a name to the container
- `docker start CONTAINER`: Start a stopped container
- `docker stop CONTAINER`: Stop a running container
- `docker attach CONTAINER`: Attach to a running container's terminal
- `docker container ps -a`: List all containers (including stopped)
- `docker container inspect CONTAINER`: Display detailed container information

### B. Volume Management
- `docker volume create NAME`: Create a named volume
- `docker volume ls`: List volumes
- `docker volume inspect VOLUME`: Display detailed volume information
- `docker volume rm VOLUME`: Remove a volume

### C. Network Management
- `-p, --publish HOST:CONTAINER`: Publish container ports to host
- `-e, --env KEY=VALUE`: Set environment variables

### D. Image Management
- `docker build`: Build an image from a Dockerfile
  - `-t, --tag NAME:TAG`: Name and optionally tag the image
- `docker images`: List images
- `docker rmi IMAGE`: Remove an image
