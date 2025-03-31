
##  Docker Core Concepts

### Image & Container Architecture

Docker containers run from images - layered filesystem snapshots containing an application and its dependencies

**Key advantages over VMs**:
- Share host kernel (no OS duplication)
- Much faster startup time
- Less resource overhead
- Smaller disk footprint

Each layer only stores the differences from layers below it, making Docker efficient with storage. When you modify a file in a running container, it creates a copy in the top layer, leaving the original unchanged in the image layer.

### Container Lifecycle

1. **Pull Image**: `docker pull python:3.11`
2. **Create Container**: `docker create --name myapp python:3.11`  
3. **Start Container**: `docker start myapp`
4. **Execute Commands**: `docker exec -it myapp bash`
5. **Stop Container**: `docker stop myapp`
6. **Remove Container**: `docker rm myapp`

When a container is stopped, its state is lost, but filesystem changes in the writable layer persist until the container is removed

## Dockerfile Syntax & Best Practices

### Basic Structure

```dockerfile
FROM base-image:tag
WORKDIR /app
COPY . .
RUN command
EXPOSE port
CMD ["executable", "param1", "param2"]
```

### Detailed Dockerfile Instructions

- **FROM**: Specifies base image (required, must be first)
  ```dockerfile
  FROM python:3.11-slim
  ```

- **WORKDIR**: Sets working directory for subsequent instructions
  ```dockerfile
  WORKDIR /app
  ```

- **COPY**: Copies files from host to container
  ```dockerfile
  # Basic syntax
  COPY source destination
  
  # Copy specific files
  COPY requirements.txt .
  
  # Copy entire directory
  COPY . .
  
  # Copy with permissions
  COPY --chown=user:group source destination
  ```

- **ADD**: Similar to COPY but with tar extraction and URL support
  ```dockerfile
  # Extract archive automatically
  ADD archive.tar.gz /app/
  
  # Download from URL
  ADD https://example.com/file.txt /app/
  ```

- **RUN**: Executes commands in a new layer
  ```dockerfile
  # Shell form
  RUN apt-get update && apt-get install -y package
  
  # Exec form
  RUN ["executable", "param1", "param2"]
  
  # Multi-line for readability
  RUN apt-get update && \
      apt-get install -y \
        package1 \
        package2 \
      && rm -rf /var/lib/apt/lists/*
  ```

- **ENV**: Sets environment variables
  ```dockerfile
  ENV NODE_ENV=production
  ENV PATH="/app/node_modules/.bin:${PATH}"
  ```

- **ARG**: Defines build-time variables
  ```dockerfile
  ARG VERSION=latest
  FROM base:${VERSION}
  ```

- **EXPOSE**: Documents ports the container will listen on
  ```dockerfile
  EXPOSE 8000
  ```

- **VOLUME**: Creates a mount point for external volumes
  ```dockerfile
  VOLUME /data
  ```

- **USER**: Sets the user for subsequent commands
  ```dockerfile
  USER appuser
  ```

- **CMD**: Default command to run when container starts (only one per Dockerfile)
  ```dockerfile
  # Exec form (preferred)
  CMD ["gunicorn", "--bind", "0.0.0.0:8000", "app:app"]
  
  # Shell form
  CMD gunicorn --bind 0.0.0.0:8000 app:app
  ```

- **ENTRYPOINT**: Makes container run as executable
  ```dockerfile
  ENTRYPOINT ["python", "app.py"]
  # With CMD, parameters can be overridden
  CMD ["--debug"]
  ```

### Building Images Efficiently

Optimizing Dockerfiles for build speed and image size:

1. **Order matters** - Place infrequently changing instructions early
   ```dockerfile
   # Better - dependencies rarely change
   COPY requirements.txt .
   RUN pip install -r requirements.txt
   COPY . .
   
   # Worse - rebuilds dependencies when code changes
   COPY . .
   RUN pip install -r requirements.txt
   ```

2. **Use .dockerignore** to exclude unnecessary files
   ```
   # Example .dockerignore
   .git
   __pycache__/
   *.pyc
   node_modules/
   npm-debug.log
   Dockerfile
   .dockerignore
   ```

3. **Minimize layers** by combining related commands
   ```dockerfile
   # Good
   RUN apt-get update && \
       apt-get install -y package && \
       rm -rf /var/lib/apt/lists/*
   
   # Bad (creates 3 layers)
   RUN apt-get update
   RUN apt-get install -y package
   RUN rm -rf /var/lib/apt/lists/*
   ```

4. **Use multi-stage builds** for smaller production images
   ```dockerfile
   # Build stage
   FROM node:18 AS build
   WORKDIR /app
   COPY package*.json ./
   RUN npm ci
   COPY . .
   RUN npm run build
   
   # Production stage
   FROM nginx:alpine
   COPY --from=build /app/build /usr/share/nginx/html
   EXPOSE 80
   CMD ["nginx", "-g", "daemon off;"]
   ```

## Real-World Dockerfiles

### Python (Django) Backend

```dockerfile
FROM python:3.11-slim

# Set environment variables
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1

# Install system dependencies
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    build-essential \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# Set working directory
WORKDIR /app

# Install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy project files
COPY . .

# Run collectstatic (if applicable)
RUN python manage.py collectstatic --no-input

# Set up a non-root user (security best practice)
RUN useradd -m appuser
RUN chown -R appuser:appuser /app
USER appuser

# Expose the port
EXPOSE 8000

# Run the application
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "project.wsgi:application"]
```

### Node.js (React) Frontend - Development

```dockerfile
FROM node:18-alpine

# Set working directory
WORKDIR /app

# Install dependencies
COPY package.json package-lock.json ./
RUN npm ci

# Copy source code
COPY . .

# Expose development server port
EXPOSE 3000

# Start development server with hot reloading
CMD ["npm", "start"]
```

### Node.js (React) Frontend - Production Build

```dockerfile
# Build stage
FROM node:18-alpine AS build

WORKDIR /app

# Install dependencies
COPY package.json package-lock.json ./
RUN npm ci

# Copy source and build
COPY . .
RUN npm run build

# Production stage
FROM nginx:alpine

# Copy build from previous stage
COPY --from=build /app/build /usr/share/nginx/html

# Copy custom nginx config if needed
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

## Data Persistence Strategies

### Understanding Docker's Storage Types

1. **Container read-write layer**: Ephemeral, lost when container is removed
2. **Volumes**: Managed by Docker, persistent, best performance
3. **Bind mounts**: Connect host directory to container, good for development
4. **tmpfs mounts**: Stored in memory, never on disk, for sensitive data

### Implementing Volumes

**Creating and using volumes**:

```bash
# Create a named volume
docker volume create my_db_data

# Run container with volume
docker run -v my_db_data:/var/lib/postgresql/data postgres:15

# Inspect volume
docker volume inspect my_db_data

# List all volumes
docker volume ls

# Remove volume
docker volume rm my_db_data
```

**Volume syntax**:
- Long form: `--mount source=my_volume,destination=/path/in/container`
- Short form: `-v my_volume:/path/in/container`

**Docker Compose volume configuration**:
```yaml
services:
  db:
    image: postgres:15
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init-scripts:/docker-entrypoint-initdb.d

volumes:
  postgres_data:
    # This creates a named volume
```

### Using Bind Mounts for Development

**Command line bind mount**:
```bash
# Mount current directory to /app
docker run -v $(pwd):/app python:3.11

# Read-only bind mount
docker run -v $(pwd):/app:ro python:3.11
```

**Docker Compose bind mounts**:
```yaml
services:
  backend:
    build: ./backend
    volumes:
      - ./backend:/app
      - /app/node_modules  # Anonymous volume to prevent overwrite
```

### Data Persistence Best Practices

1. **Database storage**: Always use volumes
   ```yaml
   volumes:
     - db_data:/var/lib/postgresql/data
   ```

2. **Uploaded files**: Use volumes with appropriate permissions
   ```yaml
   volumes:
     - uploads_data:/app/media
   ```

3. **Log files**: Consider dedicated volume or host bind mount
   ```yaml
   volumes:
     - ./logs:/app/logs
   ```

4. **Configuration**: Use bind mounts for development, volumes for production
   ```yaml
   volumes:
     - ./config.dev.json:/app/config.json  # Development
     - config_vol:/app/config  # Production
   ```

5. **Anonymous volumes**: Protect specific paths from being overwritten
   ```yaml
   volumes:
     - ./app:/app
     - /app/node_modules  # Protects node_modules from host overwrites
   ```

## Multi-Container Applications with Docker Compose

### Detailed `docker-compose.yml` Structure

```yaml
version: '3.8'  # Compose file format version

services:
  # Frontend service
  frontend:
    build:  # Build configuration
      context: ./frontend  # Build context directory
      dockerfile: Dockerfile  # Optional if named "Dockerfile"
      args:  # Build arguments
        - NODE_ENV=development
    image: myapp/frontend:latest  # Custom image name
    container_name: myapp-frontend  # Custom container name
    ports:
      - "3000:3000"  # HOST:CONTAINER
    volumes:
      - ./frontend:/app  # Bind mount for development
      - /app/node_modules  # Anonymous volume
    environment:
      - REACT_APP_API_URL=http://localhost:8000
    env_file:  # Load environment from file
      - ./frontend/.env
    depends_on:  # Startup order
      - backend
    networks:
      - frontend-network
      - backend-network
    restart: unless-stopped  # Restart policy

  # Backend service
  backend:
    build: ./backend
    ports:
      - "8000:8000"
    volumes:
      - ./backend:/app
      - static-files:/app/static
      - media-files:/app/media
    environment:
      - DEBUG=True
      - DATABASE_URL=postgresql://postgres:postgres@db:5432/myapp
    depends_on:
      - db
    networks:
      - backend-network
    healthcheck:  # Container health check
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  # Database service
  db:
    image: postgres:15-alpine
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./db/init:/docker-entrypoint-initdb.d
    environment:
      - POSTGRES_PASSWORD=postgres
      - POSTGRES_USER=postgres
      - POSTGRES_DB=myapp
    ports:
      - "5432:5432"
    networks:
      - backend-network
    restart: always

# Volumes definition
volumes:
  postgres_data:  # Managed by Docker
  static-files:
  media-files:
    driver: local  # Optional volume driver
    driver_opts:  # Driver-specific options
      type: none
      device: ${PWD}/media
      o: bind

# Networks definition
networks:
  frontend-network:
  backend-network:
    driver: bridge  # Network driver
    ipam:  # IP address management
      driver: default
      config:
        - subnet: 172.28.0.0/16
```

### Advanced Docker Compose Features

#### Environment Variables

```yaml
services:
  app:
    environment:
      - SIMPLE_VAR=value
      - VAR_FROM_SHELL=${SHELL_VAR}
    env_file:
      - ./.env.dev
```

#### Health Checks

```yaml
services:
  web:
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
```

#### Dependency Conditions

```yaml
services:
  app:
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
```

#### Profiles for Different Environments

```yaml
services:
  app:
    # Always starts
  
  db:
    # Always starts
  
  dev-tools:
    profiles: ["development"]
  
  monitoring:
    profiles: ["production"]

# Start only production services
# docker-compose --profile production up
```

##  Container Networking

### Network Types

- **bridge**: Default, connects containers on same Docker host
- **host**: Uses host's networking directly
- **overlay**: Connects containers across multiple Docker hosts
- **macvlan**: Assigns MAC address to container (appears as physical device)
- **none**: Disables networking

### Container Communication Patterns

1. **Service discovery in Docker Compose**:
   - Containers can reach each other by service name
   - Example: backend can connect to `db:5432`

2. **Port mapping to host**:
   ```yaml
   services:
     web:
       ports:
         - "8080:80"  # HOST:CONTAINER
   ```

3. **Internal networks only**:
   ```yaml
   services:
     db:
       expose:
         - "5432"  # Only exposed to other containers, not host
   ```

4. **Multiple networks**:
   ```yaml
   services:
     frontend:
       networks:
         - frontend-net
         - backend-net
     backend:
       networks:
         - backend-net
     db:
       networks:
         - backend-net

   networks:
     frontend-net:
     backend-net:
   ```

### Common Network Configurations

**Web application with API and database**:
```yaml
services:
  nginx:
    ports:
      - "80:80"
    networks:
      - frontend
  
  api:
    expose:
      - "8000"
    networks:
      - frontend
      - backend
  
  db:
    expose:
      - "5432"
    networks:
      - backend

networks:
  frontend:
  backend:
    internal: true  # No external connectivity
```

## Running and Managing Containers

### Detailed Docker Run Options

```bash
# Basic run with port mapping
docker run -p 8000:8000 my-image

# Background (detached) mode
docker run -d --name mycontainer my-image

# Environment variables
docker run -e VAR1=value1 -e VAR2=value2 my-image

# Memory limits
docker run --memory="512m" --memory-swap="1g" my-image

# CPU limits
docker run --cpus="1.5" my-image

# Restart policies
docker run --restart=always my-image
# Options: no, on-failure[:max-retries], always, unless-stopped

# Networking
docker run --network=my-network my-image

# User
docker run --user="1000:1000" my-image

# Working directory
docker run --workdir="/app" my-image

# Volumes
docker run -v my-volume:/app/data -v /host/path:/app/config my-image

# Cleanup on exit
docker run --rm my-image
```

### Container Lifecycle Management

```bash
# List containers
docker ps  # Running only
docker ps -a  # All containers

# Start/stop containers
docker start mycontainer
docker stop mycontainer
docker restart mycontainer

# Pause/unpause containers
docker pause mycontainer
docker unpause mycontainer

# Container information
docker inspect mycontainer
docker logs mycontainer
docker stats mycontainer

# Execute commands in container
docker exec -it mycontainer bash

# Update container configuration
docker update --cpus="2" --memory="1g" mycontainer

# Remove container
docker rm mycontainer
docker rm -f mycontainer  # Force remove running container
```

### Docker Compose Commands

```bash
# Start services
docker-compose up
docker-compose up -d  # Detached mode
docker-compose up --build  # Rebuild images
docker-compose up service1 service2  # Specific services

# View status
docker-compose ps

# View logs
docker-compose logs
docker-compose logs -f  # Follow
docker-compose logs service1  # Specific service

# Execute commands
docker-compose exec service1 command
docker-compose exec db psql -U postgres

# Stop services
docker-compose down  # Stop and remove containers
docker-compose down -v  # Also remove volumes
docker-compose down --rmi all  # Also remove images

# Scale services
docker-compose up -d --scale worker=3
```

## Troubleshooting and Debugging

### Common Issues and Solutions

#### 1. Container exits immediately

**Symptoms**:
- Container starts and immediately stops
- `docker ps` doesn't show container running

**Causes & Solutions**:
- **No foreground process**: Make sure CMD runs in foreground
  ```dockerfile
  # Wrong: process runs in background
  CMD nginx
  
  # Correct: keeps process in foreground
  CMD ["nginx", "-g", "daemon off;"]
  ```
- **Application crash**: Check logs
  ```bash
  docker logs container_id
  ```
- **Missing dependencies**: Verify all required files are included
  ```bash
  docker exec -it container_id sh  # Check if files exist
  ```

#### 2. Can't connect to container service

**Symptoms**:
- Service running in container but not accessible

**Causes & Solutions**:
- **Port not exposed**: Add EXPOSE in Dockerfile and -p in run command
- **Service bound to 127.0.0.1**: Bind to 0.0.0.0 instead
  ```python
  # Wrong
  app.run(host='127.0.0.1', port=8000)
  
  # Correct
  app.run(host='0.0.0.0', port=8000)
  ```
- **Firewall issues**: Check firewall settings on host

#### 3. Volume mount issues

**Symptoms**:
- Files not visible in container
- Changes not persisting

**Causes & Solutions**:
- **Path issues**: Verify paths are correct
  ```bash
  # Check volume mounts
  docker inspect container_id | grep -A 10 Mounts
  ```
- **Permissions**: Check user/group permissions
  ```bash
  docker exec -it container_id ls -la /mount/point
  ```
- **Cached volumes**: For Docker Desktop, check file sharing settings

#### 4. Performance issues

**Symptoms**:
- Containers running slowly
- High CPU or memory usage

**Causes & Solutions**:
- **Resource limits**: Set appropriate memory/CPU limits
  ```bash
  docker run --memory="2g" --cpus="2" my-image
  ```
- **Volume I/O**: For development on Mac/Windows, limit file watching
  ```yaml
  volumes:
    # Instead of mounting everything
    - ./src:/app
    # Be more specific
    - ./src/app.py:/app/app.py
  ```
- **Network traffic**: Check for unnecessary network I/O

### Interactive Debugging

```bash
# Launch shell in running container
docker exec -it container_id bash

# Launch shell in new container with image
docker run -it --rm my-image bash

# View real-time resource usage
docker stats container_id

# Access logs with timestamps
docker logs --timestamps container_id

# Follow logs in real-time
docker logs -f container_id

# Inspect networks
docker network inspect network_name
```

## Production Best Practices

### Security Considerations

1. **Use specific versions** - Never use `latest` tag in production
   ```dockerfile
   # Bad
   FROM python:latest
   
   # Good
   FROM python:3.11.4-slim
   ```

2. **Don't run as root** - Create and use non-root user
   ```dockerfile
   RUN useradd -m appuser
   USER appuser
   ```

3. **Minimize image size** - Use slim/alpine base images when possible
   ```dockerfile
   # Instead of
   FROM python:3.11
   
   # Consider
   FROM python:3.11-slim
   # or
   FROM python:3.11-alpine  # Be aware of musl libc differences
   ```

4. **Scan for vulnerabilities**
   ```bash
   docker scan my-image
   ```

5. **Use secrets management** - Don't hardcode secrets
   ```yaml
   services:
     app:
       secrets:
         - db_password
   
   secrets:
     db_password:
       file: ./secrets/db_password.txt
   ```

### Performance Optimization

1. **Multi-stage builds** for smaller images
2. **Set resource limits** to prevent resource starvation
   ```yaml
   services:
     app:
       deploy:
         resources:
           limits:
             cpus: '0.50'
             memory: 512M
   ```
3. **Use appropriate base images** - Alpine for size, Debian for compatibility
4. **Optimize layer caching** - Put infrequently changed items first
5. **Use .dockerignore** file to exclude unnecessary files

### Monitoring and Logging

1. **Structured logging** - Use JSON format for logs
2. **Log to stdout/stderr** - Let Docker handle log collection
3. **Health checks** - Implement and use container health checks
   ```yaml
   healthcheck:
     test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
     interval: 30s
     timeout: 10s
     retries: 3
   ```
4. **Container metrics** - Monitor with Docker stats or external tools
   ```bash
   docker stats
   ```

## Docker in CI/CD Pipelines

### GitHub Actions Example

```yaml
name: Docker CI

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2
      
      - name: Build and test
        uses: docker/build-push-action@v4
        with:
          context: .
          push: false
          load: true
          tags: myapp:test
          cache-from: type=gha
          cache-to: type=gha,mode=max
      
      - name: Run tests
        run: docker run --rm myapp:test pytest
      
      - name: Login to Docker Hub
        if: github.event_name != 'pull_request'
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKER_HUB_USERNAME }}
          password: ${{ secrets.DOCKER_HUB_TOKEN }}
      
      - name: Build and push
        if: github.event_name != 'pull_request'
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: |
            username/myapp:latest
            username/myapp:${{ github.sha }}
          cache-from: type=gha
```
