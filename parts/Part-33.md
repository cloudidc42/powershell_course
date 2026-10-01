# Part 33: Docker และ Containers

> **ระดับ**: 🟠 Advanced | **เวลา**: ~4 ชั่วขนึ่ง

---

## 1. Docker Commands

```powershell
# ตรวจสอบ Docker
docker version
docker info
docker system df  # ดูพื้นที่ใช้

# Images
docker images
docker pull mcr.microsoft.com/powershell:latest
docker image inspect mcr.microsoft.com/powershell:latest
docker rmi <image-id>

# Containers
docker ps           # running
docker ps -a        # all
docker run -it --rm mcr.microsoft.com/powershell:latest pwsh
docker run -d --name myapp -p 8080:80 nginx
docker stop myapp
docker start myapp
docker rm myapp

# Exec into container
docker exec -it myapp /bin/bash
docker exec myapp pwsh -Command 'Get-Process'

# Logs
docker logs myapp --tail 50 -f

# Volumes
docker volume create mydata
docker run -v mydata:/data -v C:\scripts:/scripts nginx

# Network
docker network ls
docker network create mynet
docker run --network mynet --name db mysql
```

---

## 2. Dockerfile

```dockerfile
# PowerShell app Dockerfile
FROM mcr.microsoft.com/powershell:latest

WORKDIR /app

# Copy scripts
COPY *.ps1 ./
COPY modules/ ./modules/

# Install modules
RUN pwsh -c 'Install-Module PSScriptAnalyzer -Force -Scope CurrentUser'

# Environment variables
ENV APP_ENV=production
ENV LOG_LEVEL=info

# Expose port
EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=5s \
    CMD pwsh -c 'Invoke-WebRequest http://localhost:8080/health -UseBasicParsing' \
    || exit 1

# Entry point
CMD ["pwsh", "-File", "/app/server.ps1"]
```

```powershell
# Build และ push image
docker build -t myapp:1.0 -t myapp:latest .
docker build --no-cache -t myapp:1.0 .

# Tag และ push ไป registry
docker tag myapp:1.0 registry.example.com/myapp:1.0
docker push registry.example.com/myapp:1.0
```

---

## 3. Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  api:
    build: .
    image: myapp-api:latest
    ports:
      - '8080:8080'
    environment:
      - DB_HOST=db
      - DB_PORT=5432
    depends_on:
      - db
    restart: unless-stopped
    volumes:
      - logs:/app/logs
    healthcheck:
      test: ['CMD', 'pwsh', '-c', 'Invoke-WebRequest http://localhost:8080/health -UseBasicParsing']
      interval: 30s
      timeout: 5s
      retries: 3
  
  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: admin
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    volumes:
      - pgdata:/var/lib/postgresql/data
    secrets:
      - db_password

  nginx:
    image: nginx:alpine
    ports:
      - '80:80'
      - '443:443'
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - certs:/etc/ssl/certs
    depends_on:
      - api

volumes:
  pgdata:
  logs:
  certs:

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

```powershell
# Compose commands
docker compose up -d           # start detached
docker compose up --build -d   # rebuild
docker compose down            # stop + remove
docker compose down -v         # + remove volumes
docker compose logs -f api     # follow logs
docker compose ps
docker compose exec api pwsh
```

---

## 4. PowerShell Docker Automation

```powershell
# Deploy script
function Deploy-DockerApp {
    param(
        [string]$AppName,
        [string]$Tag = 'latest',
        [string]$Registry = 'registry.example.com',
        [string]$ComposeFile = 'docker-compose.yml'
    )
    
    Write-Host "Deploying $AppName:$Tag" -ForegroundColor Cyan
    
    # Pull latest image
    docker pull "$Registry/${AppName}:${Tag}"
    if ($LASTEXITCODE -ne 0) { throw "Pull failed" }
    
    # Update service
    $env:APP_VERSION = $Tag
    docker compose -f $ComposeFile up -d --no-deps $AppName
    if ($LASTEXITCODE -ne 0) { throw "Deploy failed" }
    
    # Health check
    $tries = 0
    do {
        Start-Sleep 5
        $health = docker inspect --format '{{.State.Health.Status}}' $AppName
        $tries++
        Write-Host "  Health: $health (attempt $tries)"
    } while ($health -ne 'healthy' -and $tries -lt 12)
    
    if ($health -ne 'healthy') {
        docker rollback $AppName  # or roll back manually
        throw "Deploy failed: unhealthy after 60s"
    }
    
    Write-Host "Deploy successful!" -ForegroundColor Green
}
```

---

**ก่อนหน้า ← [Part 32](Part-32.md) | ต่อไป → [Part 34: CI/CD DevOps](Part-34.md)**
