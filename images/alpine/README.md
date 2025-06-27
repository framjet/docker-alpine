# framjet/alpine

[![Docker Hub](https://img.shields.io/docker/pulls/framjet/alpine)](https://hub.docker.com/r/framjet/alpine)
[![Docker Image Size](https://img.shields.io/docker/image-size/framjet/alpine/latest)](https://hub.docker.com/r/framjet/alpine)

Enhanced Alpine Linux Docker image with universal entrypoint system and built-in service dependency management. Perfect foundation for building robust, production-ready containers with automatic initialization and dependency orchestration.

## 🚀 Features

- **🪶 Lightweight**: Built on Alpine Linux (~14MB total)
- **🚀 Universal Entrypoint**: Flexible entrypoint system with custom script execution
- **⏱️ Dependency Management**: Built-in wait4x for service dependencies
- **🔧 Customizable**: Support for `.sh` and `.envsh` initialization scripts
- **📊 Structured Logging**: Colored, configurable logging system
- **🐳 Container-Ready**: Optimized for containerized environments
- **🏗️ Multi-Architecture**: Supports `linux/amd64` and `linux/arm64`
- **🔄 Auto-Updated**: Automatically tracks Alpine upstream releases

## 🚀 Quick Start

### Basic Usage

```bash
# Direct execution
docker run --rm framjet/alpine echo "Hello World"

# Interactive shell
docker run --rm -it framjet/alpine sh
```

### With Service Dependencies

```bash
# Wait for services before starting
docker run --rm \
  -e WAIT_FOR_DB=postgres:5432#60s \
  -e WAIT_FOR_CACHE=redis:6379#30s \
  -e DOCKER_ENTRYPOINT_COMMANDS=sh \
  framjet/alpine sh -c "echo 'Services ready!'"
```

### Docker Compose

```yaml
version: '3.8'
services:
  app:
    image: framjet/alpine
    command: ["sh", "-c", "echo 'App started!' && sleep 300"]
    environment:
      - WAIT_FOR_DB=postgres:5432#60s
      - WAIT_FOR_CACHE=redis:6379#30s
      - DOCKER_ENTRYPOINT_COMMANDS=sh
    depends_on:
      - postgres
      - redis
    
  postgres:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: password
      
  redis:
    image: redis:7-alpine
```

## ⚙️ Entrypoint System

### Command Triggers

The entrypoint system activates based on the `DOCKER_ENTRYPOINT_COMMANDS` environment variable:

```bash
# Single command
DOCKER_ENTRYPOINT_COMMANDS="sh"

# Multiple commands (comma-separated)
DOCKER_ENTRYPOINT_COMMANDS="sh,bash,python,myapp"

# Only run when specific commands are executed
docker run -e DOCKER_ENTRYPOINT_COMMANDS="myapp" framjet/alpine myapp
```

### Initialization Scripts Directory

Scripts in `/docker-entrypoint.d/` are executed automatically when the entrypoint triggers:

| Extension | Behavior                     | Use Case                                     |
|-----------|------------------------------|----------------------------------------------|
| `*.sh`    | Executed as separate process | Setup, configuration, service initialization |
| `*.envsh` | Sourced into current shell   | Environment variable setup, aliases          |

### Execution Order

Scripts are executed in lexicographical order (using `sort -V`):

```
/docker-entrypoint.d/
├── 01-environment.envsh     # Environment setup (sourced)
├── 10-wait-services.sh      # Service dependency waiting (built-in)
├── 20-setup-app.sh          # Application setup
├── 30-configure-logging.sh  # Logging configuration
└── 99-final-setup.sh        # Final initialization
```

## 🔄 Service Dependencies

### Environment Variable Format

```bash
# Format: WAIT_FOR_<NAME>=host:port#timeout
WAIT_FOR_DB=postgres:5432#60s
WAIT_FOR_CACHE=redis:6379#30s
WAIT_FOR_API=api-server:8080#120s

# Custom wait4x commands
WAIT_CMD_FOR_<NAME>="<wait4x command>"
WAIT_CMD_FOR_HEALTH="http http://api:8080/health --timeout 30s"
WAIT_CMD_FOR_FILE="file /shared/ready.flag --timeout 60s"
```

### Supported Protocols

All wait4x protocols are supported:

```bash
# TCP connection
WAIT_FOR_DB=postgres:5432#60s

# HTTP/HTTPS endpoints
WAIT_CMD_FOR_API="http http://api:8080/health --expect-status-code 200"

# Database connections
WAIT_CMD_FOR_POSTGRES="postgresql postgres://user:pass@postgres:5432/mydb"
WAIT_CMD_FOR_MYSQL="mysql user:pass@tcp(mysql:3306)/mydb"
WAIT_CMD_FOR_MONGO="mongodb mongodb://user:pass@mongo:27017/mydb"

# File system
WAIT_CMD_FOR_CONFIG="file /config/app.conf --timeout 30s"

# Redis
WAIT_CMD_FOR_REDIS="redis redis://redis:6379"
```

### Error Handling

```bash
# The entrypoint will exit with error code 1 if any dependency fails
# Customize timeout per service for different startup characteristics

WAIT_FOR_FAST_SERVICE=cache:6379#10s      # Fast-starting service
WAIT_FOR_SLOW_SERVICE=database:5432#180s  # Slow-starting service
```

## 📊 Logging System

### Log Functions

The entrypoint provides structured logging functions:

```bash
#!/bin/sh
. /docker-entrypoint.functions

entrypoint_log "General information message"    # Green
entrypoint_info "Informational message"         # Blue  
entrypoint_warn "Warning message"               # Yellow
entrypoint_error "Error message"                # Red
```

### Configuration

```bash
# Disable all entrypoint logs
DOCKER_ENTRYPOINT_QUIET_LOGS=1

# Colors are automatically disabled when not in a terminal
docker run framjet/alpine 2>&1 | tee logfile.txt
```

### Example Output

```
myapp: Waiting for "postgres:5432" with timeout "60s"
myapp: Waiting for "redis:6379" with timeout "30s"
myapp: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
myapp: Looking for shell scripts in /docker-entrypoint.d/
myapp: Sourcing /docker-entrypoint.d/01-environment.envsh
myapp: Launching /docker-entrypoint.d/20-setup.sh
myapp: Configuration complete; ready for start up
```

## 🏗️ Building Custom Images

### Simple Application

```dockerfile
FROM framjet/alpine

# Install application runtime
RUN apk add --no-cache python3 py3-pip

# Copy application
COPY app.py /app/
COPY requirements.txt /app/

# Install dependencies
RUN pip3 install -r /app/requirements.txt

# Add initialization script
COPY setup-app.sh /docker-entrypoint.d/20-setup-app.sh

# Configure entrypoint to trigger for python commands
ENV DOCKER_ENTRYPOINT_COMMANDS="python3"

WORKDIR /app
CMD ["python3", "app.py"]
```

### Multi-Service Container

```dockerfile
FROM framjet/alpine

# Install services
RUN apk add --no-cache nginx supervisord

# Copy configurations
COPY nginx.conf /etc/nginx/
COPY supervisord.conf /etc/

# Add service setup scripts
COPY setup-nginx.sh /docker-entrypoint.d/10-setup-nginx.sh
COPY setup-supervisord.envsh /docker-entrypoint.d/05-supervisord.envsh

# Configure entrypoint
ENV DOCKER_ENTRYPOINT_COMMANDS="supervisord"

EXPOSE 80
CMD ["supervisord", "-c", "/etc/supervisord.conf"]
```

### Database Migration Container

```dockerfile
FROM framjet/alpine

# Install database client
RUN apk add --no-cache postgresql-client

# Copy migration scripts and data
COPY migrations/ /migrations/
COPY migrate.sh /usr/local/bin/

# Make migration script executable
RUN chmod +x /usr/local/bin/migrate.sh

# Configure entrypoint to wait for database
ENV DOCKER_ENTRYPOINT_COMMANDS="migrate.sh"

CMD ["migrate.sh"]
```

### Development Environment

```dockerfile
FROM framjet/alpine

# Install development tools
RUN apk add --no-cache \
    git \
    curl \
    wget \
    vim \
    bash \
    tmux

# Create development user
RUN adduser -D -s /bin/bash developer

# Add development setup
COPY dev-environment.envsh /docker-entrypoint.d/

# Configure for interactive shells
ENV DOCKER_ENTRYPOINT_COMMANDS="bash,sh"

USER developer
WORKDIR /home/developer

CMD ["bash"]
```

## 🐳 Container Patterns

### Init Container in Kubernetes

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  initContainers:
  - name: wait-for-services
    image: framjet/alpine
    command: ["sh"]
    env:
    - name: WAIT_FOR_DB
      value: "postgres-service:5432#120s"
    - name: WAIT_FOR_CACHE  
      value: "redis-service:6379#60s"
    - name: DOCKER_ENTRYPOINT_COMMANDS
      value: "sh"
  containers:
  - name: myapp
    image: myapp:latest
```

### Sidecar Pattern

```yaml
version: '3.8'
services:
  app:
    image: myapp:latest
    
  monitor:
    image: framjet/alpine
    command: |
      sh -c '
        while true; do
          if ! nc -z app 8080; then
            echo "App is down, restarting..."
            # Restart logic here
          fi
          sleep 30
        done
      '
    environment:
      - DOCKER_ENTRYPOINT_COMMANDS=sh
    depends_on:
      - app
```

### Job Runner

```dockerfile
FROM framjet/alpine

# Install job dependencies
RUN apk add --no-cache python3 py3-pip curl

# Copy job scripts
COPY job-runner.py /usr/local/bin/
COPY wait-for-work.sh /docker-entrypoint.d/10-wait-for-work.sh

# Configure for job execution
ENV DOCKER_ENTRYPOINT_COMMANDS="python3"

CMD ["python3", "/usr/local/bin/job-runner.py"]
```

## 🔧 Advanced Configuration

### Environment Variable Processing

```bash
# /docker-entrypoint.d/01-env-setup.envsh
#!/bin/sh

# This script is sourced, so variables are available to the main process
export DATABASE_URL="postgres://user:${DB_PASSWORD}@${DB_HOST:-postgres}:5432/${DB_NAME:-myapp}"
export REDIS_URL="redis://${REDIS_HOST:-redis}:6379"
export LOG_LEVEL="${LOG_LEVEL:-info}"

# Log the configuration (functions are available)
entrypoint_info "Database URL: $DATABASE_URL"
entrypoint_info "Redis URL: $REDIS_URL"
entrypoint_info "Log Level: $LOG_LEVEL"
```

### Conditional Initialization

```bash
# /docker-entrypoint.d/20-conditional-setup.sh
#!/bin/sh

. /docker-entrypoint.functions

if [ "$ENVIRONMENT" = "production" ]; then
    entrypoint_info "Configuring for production environment"
    # Production-specific setup
    apk add --no-cache ca-certificates
    
elif [ "$ENVIRONMENT" = "development" ]; then
    entrypoint_info "Configuring for development environment"
    # Development-specific setup
    apk add --no-cache curl wget
    
else
    entrypoint_warn "Unknown environment: $ENVIRONMENT, using defaults"
fi

# Always perform common setup
entrypoint_log "Common setup completed"
```

### Service Health Monitoring

```bash
# /docker-entrypoint.d/30-health-monitor.sh
#!/bin/sh

. /docker-entrypoint.functions

# Setup health monitoring in background
(
    while true; do
        sleep 30
        if ! nc -z localhost 8080; then
            entrypoint_error "Health check failed for localhost:8080"
            # Could trigger alerts or restart logic here
        fi
    done
) &

entrypoint_info "Health monitoring started"
```

## 📝 Real-World Examples

### Web Application Stack

```dockerfile
FROM framjet/alpine

# Install web server and PHP
RUN apk add --no-cache nginx php82-fpm

# Copy application code
COPY src/ /var/www/html/

# Copy configuration files
COPY nginx.conf /etc/nginx/
COPY php-fpm.conf /etc/php82/

# Add initialization scripts
COPY 10-setup-nginx.sh /docker-entrypoint.d/
COPY 20-setup-php.sh /docker-entrypoint.d/
COPY 30-setup-app.envsh /docker-entrypoint.d/

# Configure entrypoint
ENV DOCKER_ENTRYPOINT_COMMANDS="supervisord"

EXPOSE 80
CMD ["supervisord", "-c", "/etc/supervisord.conf"]
```

### Microservice with Circuit Breaker

```dockerfile
FROM framjet/alpine

# Install service runtime
RUN apk add --no-cache ca-certificates

# Copy service binary
COPY myservice /usr/local/bin/

# Add service initialization
COPY setup-circuit-breaker.envsh /docker-entrypoint.d/

# Configure dependencies
ENV WAIT_FOR_DB=postgres:5432#120s
ENV WAIT_FOR_CACHE=redis:6379#60s
ENV WAIT_CMD_FOR_UPSTREAM="http http://upstream-api:8080/health --timeout 30s"

# Configure entrypoint
ENV DOCKER_ENTRYPOINT_COMMANDS="myservice"

EXPOSE 8080
CMD ["myservice"]
```

### Data Processing Pipeline

```dockerfile
FROM framjet/alpine

# Install processing tools
RUN apk add --no-cache python3 py3-pip

# Install Python dependencies
COPY requirements.txt /tmp/
RUN pip3 install -r /tmp/requirements.txt

# Copy processing scripts
COPY process-data.py /usr/local/bin/
COPY wait-for-input.sh /docker-entrypoint.d/10-wait-for-input.sh

# Configure to wait for input data and database
ENV WAIT_CMD_FOR_INPUT="file /data/input/ready.flag --timeout 300s"
ENV WAIT_FOR_DB=postgres:5432#180s

ENV DOCKER_ENTRYPOINT_COMMANDS="python3"

CMD ["python3", "/usr/local/bin/process-data.py"]
```

## 🔍 Troubleshooting

### Debug Entrypoint Execution

```bash
# Enable verbose logging
docker run -it \
  -e DOCKER_ENTRYPOINT_COMMANDS=sh \
  framjet/alpine sh -x /docker-entrypoint.sh sh
```

### Test Service Dependencies

```bash
# Test specific service dependency
docker run --rm \
  --network myapp_network \
  -e WAIT_FOR_TEST=postgres:5432#10s \
  -e DOCKER_ENTRYPOINT_COMMANDS=sh \
  framjet/alpine sh
```

### Inspect Initialization Scripts

```bash
# List all initialization scripts
docker run --rm framjet/alpine find /docker-entrypoint.d/ -type f -ls

# Test script execution manually
docker run --rm -it framjet/alpine sh -c "
  . /docker-entrypoint.functions
  entrypoint_log 'Testing logging functions'
"
```

## 🤝 Related Images

- [`alpine:latest`](https://hub.docker.com/_/alpine) - Original Alpine Linux image
- [`framjet/wait4x`](https://github.com/framjet/docker-wait4x) - Standalone wait4x utility
- [`framjet/lighttpd`](https://github.com/framjet/docker-lighttpd) - Lighttpd web server

## 📄 License

MIT License - see [LICENSE](../../LICENSE) for details.

## 🆘 Support

- 🐛 [Report Issues](https://github.com/framjet/docker-alpine/issues)
- 💬 [Discussions](https://github.com/framjet/docker-alpine/discussions)
- 📖 [Alpine Documentation](https://wiki.alpinelinux.org/)
- 🐳 [Docker Hub](https://hub.docker.com/r/framjet/alpine)
---

**Made with ❤️ by [FramJet](https://github.com/framjet)**
