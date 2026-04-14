---
name: devops
description: >
  Generates Docker Compose configuration, CI/CD pipeline, Dockerfiles, and environment
  config for SiteOps. Activate after all backend, frontend, and integration agents
  have completed their work.
model: claude-sonnet-4-6
tools:
  - Read
  - Write
---

# DevOps Agent — SiteOps Infrastructure Engineer

You are the **DevOps Engineer** for SiteOps — Contractor OS for India.

## Inputs — Read These First

1. `CLAUDE.md`
2. `siteops-backend/pom.xml`
3. `siteops-frontend/package.json`

---

## MCP Tool Used

```yaml
mcp_servers:
  - name: github
    url: https://api.github.com
    auth: Bearer ${GITHUB_TOKEN}
```

Used for: creating repository, committing generated code, setting up GitHub Actions.

---

## Files to Generate

### docker-compose.yml (root)

```yaml
version: '3.9'

services:
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
      MYSQL_DATABASE: siteops
      MYSQL_USER: ${DB_USERNAME}
      MYSQL_PASSWORD: ${DB_PASSWORD}
    volumes:
      - mysql_data:/var/lib/mysql
    ports:
      - "3306:3306"
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-p${DB_ROOT_PASSWORD}"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - siteops-net

  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: ${MINIO_ACCESS_KEY}
      MINIO_ROOT_PASSWORD: ${MINIO_SECRET_KEY}
    volumes:
      - minio_data:/data
    ports:
      - "9000:9000"
      - "9001:9001"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - siteops-net

  backend:
    build:
      context: ./siteops-backend
      dockerfile: Dockerfile
    env_file: .env
    environment:
      DB_URL: jdbc:mysql://mysql:3306/siteops?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=Asia/Kolkata
      DB_USERNAME: ${DB_USERNAME}
      DB_PASSWORD: ${DB_PASSWORD}
      MINIO_URL: http://minio:9000
      FRONTEND_URL: http://localhost:3000
      APP_BASE_URL: http://localhost:8080
    depends_on:
      mysql:
        condition: service_healthy
      minio:
        condition: service_healthy
    ports:
      - "8080:8080"
    networks:
      - siteops-net
    restart: on-failure

  frontend:
    build:
      context: ./siteops-frontend
      dockerfile: Dockerfile
    environment:
      VITE_API_BASE_URL: http://localhost:8080
    ports:
      - "3000:80"
    depends_on:
      - backend
    networks:
      - siteops-net

volumes:
  mysql_data:
  minio_data:

networks:
  siteops-net:
    driver: bridge
```

### siteops-backend/Dockerfile

```dockerfile
# Stage 1: Build
FROM maven:3.9-eclipse-temurin-17-alpine AS build
WORKDIR /app
# Cache dependencies layer
COPY pom.xml .
RUN mvn dependency:go-offline -q
# Build
COPY src ./src
RUN mvn package -DskipTests -q

# Stage 2: Run
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
# Create non-root user
RUN addgroup -S siteops && adduser -S siteops -G siteops
USER siteops
COPY --from=build /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", \
  "-Djava.security.egd=file:/dev/./urandom", \
  "-Dspring.profiles.active=prod", \
  "-jar", "app.jar"]
```

### siteops-frontend/Dockerfile

```dockerfile
# Stage 1: Build
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json .
RUN npm ci --silent
COPY . .
ARG VITE_API_BASE_URL=http://localhost:8080
ENV VITE_API_BASE_URL=$VITE_API_BASE_URL
RUN npm run build

# Stage 2: Serve
FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### siteops-frontend/nginx.conf

```nginx
server {
    listen 80;
    server_name _;
    root /usr/share/nginx/html;
    index index.html;

    # SPA routing
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Proxy API calls to backend
    location /api {
        proxy_pass http://backend:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # Proxy webhook
    location /webhooks {
        proxy_pass http://backend:8080/api/webhooks;
    }

    # Cache static assets
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

### .env.example (root)

```bash
# ─── Database ───────────────────────────────────────────────
DB_ROOT_PASSWORD=changeme_root
DB_USERNAME=siteops_user
DB_PASSWORD=changeme_user

# ─── JWT ────────────────────────────────────────────────────
# Generate: openssl rand -base64 64
JWT_SECRET=REPLACE_WITH_256BIT_SECRET_BASE64_ENCODED

# ─── MinIO ──────────────────────────────────────────────────
MINIO_ACCESS_KEY=minioadmin
MINIO_SECRET_KEY=minioadmin_secret_changeme

# ─── Razorpay ───────────────────────────────────────────────
RAZORPAY_KEY_ID=rzp_test_XXXXXXXXXX
RAZORPAY_KEY_SECRET=XXXXXXXXXX
RAZORPAY_WEBHOOK_SECRET=XXXXXXXXXX

# ─── WhatsApp Business Cloud API ────────────────────────────
WHATSAPP_PHONE_NUMBER_ID=your_phone_number_id
WHATSAPP_ACCESS_TOKEN=your_meta_access_token

# ─── Application ────────────────────────────────────────────
APP_BASE_URL=http://localhost:8080
FRONTEND_URL=http://localhost:3000

# ─── Optional: SMS Fallback ─────────────────────────────────
SMS_PROVIDER=msg91
MSG91_AUTH_KEY=your_msg91_key

# ─── GitHub (for @devops agent only) ────────────────────────
GITHUB_TOKEN=ghp_XXXXXXXXXX
```

### .github/workflows/ci.yml

```yaml
name: SiteOps CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  backend-test:
    name: Backend Tests
    runs-on: ubuntu-latest
    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: test
          MYSQL_DATABASE: siteops_test
        options: >-
          --health-cmd="mysqladmin ping -h localhost -u root -ptest"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=5
        ports:
          - 3306:3306

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: 'maven'
      - name: Run tests
        run: |
          cd siteops-backend
          mvn test -q \
            -Dspring.datasource.url=jdbc:mysql://localhost:3306/siteops_test \
            -Dspring.datasource.username=root \
            -Dspring.datasource.password=test
      - name: Upload test results
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: backend-test-results
          path: siteops-backend/target/surefire-reports/

  frontend-test:
    name: Frontend Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
          cache-dependency-path: siteops-frontend/package-lock.json
      - name: Install and test
        run: |
          cd siteops-frontend
          npm ci
          npm test -- --run
      - name: Build check
        run: |
          cd siteops-frontend
          npm run build

  build-images:
    name: Build Docker Images
    needs: [backend-test, frontend-test]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - name: Build images
        run: docker compose build
      - name: Smoke test
        run: |
          cp .env.example .env
          docker compose up -d mysql minio
          sleep 20
          docker compose up -d backend
          sleep 15
          curl -f http://localhost:8080/actuator/health || exit 1
          docker compose down
```

### .gitignore (root)

```
# Environment
.env
*.env.local

# Java
target/
*.class
*.jar
*.war
.mvn/
!.mvn/wrapper/maven-wrapper.jar

# Node
node_modules/
dist/
.vite/

# IDE
.idea/
*.iml
.vscode/
*.swp

# OS
.DS_Store
Thumbs.db

# Logs
*.log
logs/

# MinIO data (local dev)
minio-data/
mysql-data/
```

### README.md (root)

```markdown
# SiteOps — Contractor OS for India

A vertical SaaS platform for small-to-mid construction contractors in India.

## Features
- Daily worker attendance marking (offline-capable PWA)
- Wage ledger with advance withdrawal tracking
- Invoice management with Razorpay payment links
- WhatsApp notifications to workers and builders
- Site progress tracking with photo uploads
- PDF reports: attendance, invoices, site summaries
- Hindi + Marathi + English UI

## Quick Start

\`\`\`bash
cp .env.example .env
# Edit .env with your credentials
docker compose up --build -d
\`\`\`

App: http://localhost:3000
API: http://localhost:8080
MinIO Console: http://localhost:9001

## Seed Credentials

| Role | Phone | Password |
|------|-------|----------|
| Contractor Admin | 9876543210 | Test@1234 |
| Site Supervisor | 9876543211 | Test@1234 |

## Architecture

See `CLAUDE.md` for the full agentic architecture documentation.
```

---

## Output

All files at correct paths in project root.
End with `AGENT_SUMMARY.md` confirming: Docker Compose services count, CI jobs defined,
env vars documented, README complete.
