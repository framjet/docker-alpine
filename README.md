# FramJet Alpine Docker Images

[![Build Status](https://github.com/framjet/docker-alpine/actions/workflows/build-images.yaml/badge.svg)](https://github.com/framjet/docker-alpine/actions/workflows/build-images.yaml)
[![Docker Hub](https://img.shields.io/badge/Docker%20Hub-framjet-blue)](https://hub.docker.com/u/framjet)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

Enhanced Alpine Linux Docker images with universal entrypoint system and built-in service dependency management. Designed as a foundation for building robust, production-ready containers.

## 🚀 Available Images

| Image                                                       | Description                            | Size  | Use Case                                   |
|-------------------------------------------------------------|----------------------------------------|-------|--------------------------------------------|
| [`framjet/alpine`](https://hub.docker.com/r/framjet/alpine) | Enhanced Alpine with entrypoint system | ~14MB | Base image, init containers, custom builds |

## ⭐ Key Features

- **🪶 Lightweight**: Built on Alpine Linux for minimal footprint
- **🚀 Universal Entrypoint**: Flexible entrypoint system with custom script support
- **⏱️ Dependency Management**: Built-in wait4x for service dependencies
- **🔧 Customizable**: Support for initialization scripts and environment configuration
- **🐳 Container-Ready**: Perfect foundation for production containers
- **📊 Monitoring**: Colored logging with configurable output levels
- **🏗️ Multi-Architecture**: Supports `linux/amd64` and `linux/arm64`
- **🔄 Auto-Updated**: Automatically tracks Alpine upstream releases

## 🚀 Quick Start

### Basic Usage

```bash
# Use as base image
FROM framjet/alpine
CMD ["echo", "Hello World"]

# Direct execution
docker run --rm framjet/alpine echo "Hello World"
```

### With Service Dependencies

```bash
# Wait for services before starting
docker run --rm \
  -e WAIT_FOR_DB=postgres:5432#60s \
  -e WAIT_FOR_CACHE=redis:6379#30s \
  framjet/alpine echo "Services are ready!"
```

### Docker Compose

```yaml
version: '3.8'
services:
  app:
    image: framjet/alpine
    command: ["sh", "-c", "echo 'App started!' && sleep 3600"]
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

## 🔧 Entrypoint System

### How It Works

The enhanced entrypoint system runs automatically when specific commands are executed, controlled by the `DOCKER_ENTRYPOINT_COMMANDS` environment variable.

```bash
# Configure which commands trigger entrypoint processing
docker run -e DOCKER_ENTRYPOINT_COMMANDS="sh,bash,my-app" framjet/alpine sh
```

### Custom Initialization Scripts

Add scripts to `/docker-entrypoint.d/` for automatic execution:

```dockerfile
FROM framjet/alpine

# Add custom initialization scripts
COPY my-init.sh /docker-entrypoint.d/10-my-init.sh
COPY environment.envsh /docker-entrypoint.d/05-environment.envsh

# Set commands that trigger entrypoint
ENV DOCKER_ENTRYPOINT_COMMANDS="my-app"

CMD ["my-app"]
```

Supported script types:
- `*.sh` - Executable shell scripts (run during startup)
- `*.envsh` - Sourced environment scripts (sourced into shell)

## ⏱️ Service Dependencies

### Environment Variable Configuration

```bash
# Single service with timeout
WAIT_FOR_DB=postgres:5432#60s

# Multiple services
WAIT_FOR_DB=postgres:5432#60s
WAIT_FOR_CACHE=redis:6379#30s
WAIT_FOR_API=api-server:8080#45s

# Custom wait4x commands
WAIT_CMD_FOR_HEALTH="http http://api:8080/health --timeout 30s"
WAIT_CMD_FOR_FILE="file /shared/ready.flag --timeout 60s"
```

### Docker Compose Example

```yaml
version: '3.8'
services:
  web:
    build: .
    environment:
      - WAIT_FOR_DB=postgres:5432#120s
      - WAIT_FOR_CACHE=redis:6379#60s
      - DOCKER_ENTRYPOINT_COMMANDS=nginx
    depends_on:
      - postgres
      - redis
      
  worker:
    build: .
    command: ["python", "worker.py"]
    environment:
      - WAIT_FOR_DB=postgres:5432#120s
      - WAIT_CMD_FOR_MIGRATIONS="http http://web:8080/health --timeout 300s"
      - DOCKER_ENTRYPOINT_COMMANDS=python
    depends_on:
      - postgres
      - web
```

## 🏗️ Building Custom Images

### Simple Application

```dockerfile
FROM framjet/alpine

# Install application dependencies
RUN apk add --no-cache python3 py3-pip

# Copy application
COPY app.py /app/
COPY requirements.txt /app/

# Install Python dependencies
RUN pip3 install -r /app/requirements.txt

# Configure entrypoint to run for python commands
ENV DOCKER_ENTRYPOINT_COMMANDS="python3"

WORKDIR /app
CMD ["python3", "app.py"]
```

### Multi-Service Application

```dockerfile
FROM framjet/alpine

# Install services
RUN apk add --no-cache nginx php82-fpm

# Copy configurations
COPY nginx.conf /etc/nginx/
COPY php.ini /etc/php82/

# Copy initialization scripts
COPY 10-setup-nginx.sh /docker-entrypoint.d/
COPY 20-setup-php.sh /docker-entrypoint.d/

# Configure entrypoint
ENV DOCKER_ENTRYPOINT_COMMANDS="supervisord,nginx"

CMD ["supervisord", "-c", "/etc/supervisord.conf"]
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
    bash

# Setup development user
RUN adduser -D -s /bin/bash developer

# Copy development setup scripts
COPY dev-setup.envsh /docker-entrypoint.d/

# Configure entrypoint for interactive shells
ENV DOCKER_ENTRYPOINT_COMMANDS="bash,sh"

USER developer
CMD ["bash"]
```

## 📊 Logging System

### Log Levels and Colors

The entrypoint system provides colored, structured logging:

- **🟢 entrypoint_log()** - General information (green)
- **🔵 entrypoint_info()** - Informational messages (blue)
- **🟡 entrypoint_warn()** - Warnings (yellow)
- **🔴 entrypoint_error()** - Errors (red)

### Configuration

```bash
# Disable entrypoint logs completely
DOCKER_ENTRYPOINT_QUIET_LOGS=1

# Logs automatically disable colors when not running in terminal
docker run framjet/alpine 2>&1 | tee output.log
```

## 🐳 Integration Patterns

### Kubernetes Init Containers

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      initContainers:
      - name: wait-for-dependencies
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

### Helm Chart Template

```yaml
{{- if .Values.waitForDependencies.enabled }}
initContainers:
- name: wait-for-services
  image: framjet/alpine:{{ .Values.alpine.tag | default "latest" }}
  command: ["sh"]
  env:
  {{- range .Values.waitForDependencies.services }}
  - name: WAIT_FOR_{{ .name | upper }}
    value: "{{ .host }}:{{ .port }}#{{ .timeout | default "60s" }}"
  {{- end }}
  - name: DOCKER_ENTRYPOINT_COMMANDS
    value: "sh"
{{- end }}
```

### Docker Swarm Services

```yaml
version: '3.8'
services:
  app:
    image: myapp:latest
    environment:
      - WAIT_FOR_DB=postgres:5432#120s
      - DOCKER_ENTRYPOINT_COMMANDS=myapp
    depends_on:
      - postgres
    deploy:
      replicas: 3
      restart_policy:
        condition: on-failure
        delay: 5s
```

## 🔧 Advanced Configuration

### Custom Wait Strategies

```bash
# Complex dependency waiting
WAIT_CMD_FOR_DB_READY="postgresql postgres://user:pass@postgres:5432/mydb"
WAIT_CMD_FOR_MIGRATION="http http://api:8080/migration/status --expect-status-code 200"
WAIT_CMD_FOR_CACHE="redis redis://redis:6379"
```

### Environment Variable Processing

```bash
# Create environment setup script
cat > /docker-entrypoint.d/01-env-setup.envsh << 'EOF'
#!/bin/sh
# This script is sourced, so variables are available to main process

export DATABASE_URL="postgres://user:${DB_PASSWORD}@postgres:5432/mydb"
export REDIS_URL="redis://redis:6379"
export LOG_LEVEL="${LOG_LEVEL:-info}"

echo "Environment configured:"
echo "  DATABASE_URL: $DATABASE_URL"
echo "  REDIS_URL: $REDIS_URL"
echo "  LOG_LEVEL: $LOG_LEVEL"
EOF
```

### Conditional Script Execution

```bash
# Create conditional initialization script
cat > /docker-entrypoint.d/20-conditional-setup.sh << 'EOF'
#!/bin/sh

if [ "$ENVIRONMENT" = "production" ]; then
    echo "Setting up production configuration..."
    # Production-specific setup
elif [ "$ENVIRONMENT" = "development" ]; then
    echo "Setting up development configuration..."
    # Development-specific setup
fi
EOF
```

## 📝 Examples

### Microservice Base

```dockerfile
FROM framjet/alpine

# Install runtime dependencies
RUN apk add --no-cache ca-certificates tzdata

# Create app user
RUN addgroup -g 1001 app && adduser -u 1001 -G app -D app

# Copy application
COPY --chown=app:app myservice /usr/local/bin/

# Setup entrypoint for our service
ENV DOCKER_ENTRYPOINT_COMMANDS="myservice"

USER app
EXPOSE 8080

CMD ["myservice"]
```

### Database Migration Container

```dockerfile
FROM framjet/alpine

# Install migration tool
RUN apk add --no-cache postgresql-client

# Copy migration scripts
COPY migrations/ /migrations/
COPY migrate.sh /usr/local/bin/

# Setup entrypoint to wait for database
ENV DOCKER_ENTRYPOINT_COMMANDS="migrate.sh"

CMD ["migrate.sh"]
```

### Development Debugging

```dockerfile
FROM framjet/alpine

# Install debugging tools
RUN apk add --no-cache \
    strace \
    tcpdump \
    netcat-openbsd \
    curl \
    jq

# Copy debugging scripts
COPY debug-tools.sh /docker-entrypoint.d/

ENV DOCKER_ENTRYPOINT_COMMANDS="sh,bash"

CMD ["sh"]
```

## 🤝 Contributing

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 🏷️ Tags and Versioning

- `latest` - Latest Alpine release with our enhancements
- `3` - Alpine 3.x series
- Multi-architecture support: `linux/amd64`, `linux/arm64`

## 🆘 Support

- 📖 [Documentation](https://github.com/framjet/docker-alpine)
- 🐛 [Issue Tracker](https://github.com/framjet/docker-alpine/issues)
- 💬 [Discussions](https://github.com/framjet/docker-alpine/discussions)
- 🐳 [Docker Hub](https://hub.docker.com/r/framjet/alpine)

---

**Made with ❤️ by [FramJet](https://github.com/framjet)**
