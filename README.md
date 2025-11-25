# 🚀 Next.js, Prisma, and NextAuth Deployment Guide
This guide details the configuration and deployment steps for a Next.js application using Prisma for database access and NextAuth.js for authentication, packaged with Docker for a production-ready environment.
## 1. Project Setup and Configuration
This guide details the configuration and deployment steps for a Next.js application using Prisma for database access and NextAuth.js for authentication, packaged with Docker for a production-ready environment.

### 1.1 Next.js Standalone Configuration
To leverage Next.js's efficient Standalone output mode (which creates a minimal, optimized server), ensure your next.config.js is set up as follows:

```
  import type { NextConfig } from "next";
  const nextConfig: NextConfig = {
    output: "standalone",
  };
  export default nextConfig;
```

### 1.2 Environment Variables
.env 
```
  # DATABASE
  PROD_DATABASE_URL = "mysql://{user}:{password}@{host}/{db_name}"
  TEST_DATABASE_URL = "mysql://{user}:{password}@{host}/{db_name}"
  
  #AUTH
  NEXTAUTH_SECRET=generate-base64
  PROD_NEXTAUTH_URL={domain_prod}
  TEST_NEXTAUTH_URL={domain_test}
  DEV_NEXTAUTH_URL=http://localhost:3000
```
## 2.Docker Deployment
### 2.1 Multi-Stage Dockerfile
.Dockerfile
```
  # ============================
  # 1) Dependencies Stage
  # ============================
  FROM node:22-alpine AS deps
  WORKDIR /app

  # Install dependencies needed for Prisma
  RUN apk add --no-cache libc6-compat openssl

  # Copy package files
  COPY package.json package-lock.json* ./

  # Install dependencies
  RUN npm ci --legacy-peer-deps


  # ============================
  # 2) Builder Stage
  # ============================
  FROM node:22-alpine AS builder
  WORKDIR /app

  # Install dependencies needed for Prisma
  RUN apk add --no-cache openssl

  # Copy dependencies from deps stage
  COPY --from=deps /app/node_modules ./node_modules

  # Copy project files
  COPY . .

  # Generate Prisma client for Alpine Linux
  RUN npx prisma generate

  # Disable telemetry during build
  ENV NEXT_TELEMETRY_DISABLED=1

  # Build Next.js for production (turbopack is for dev only)
  RUN npx next build

  # Copy Prisma generated files to standalone output
  RUN cp -R app/generated .next/standalone/app/


  # ============================
  # 3) Production Runner Stage
  # ============================
  FROM node:22-alpine AS runner
  WORKDIR /app

  # Install OpenSSL for Prisma Client
  RUN apk add --no-cache openssl

  ENV NODE_ENV=production
  ENV NEXT_TELEMETRY_DISABLED=1

  # Create a non-root user for security
  RUN addgroup --system --gid 1001 nodejs && \
      adduser --system --uid 1001 nextjs

  # Copy Next.js standalone build (significantly smaller!)
  COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
  COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static
  COPY --from=builder --chown=nextjs:nodejs /app/public ./public

  # Copy Prisma schema and generated client
  COPY --from=builder --chown=nextjs:nodejs /app/prisma ./prisma

  # Switch to non-root user
  USER nextjs

  # Allow dynamic port configuration (default: 3006)
  ARG PORT=3006
  EXPOSE ${PORT}
  
  ENV PORT=${PORT}
  ENV HOSTNAME="0.0.0.0"
  
  CMD ["node", "server.js"]

```
### 2.2 Docker Compose
Use Docker Compose to manage the production and test service environments, including resource limits and health checks.
```
  services:
  # ============================
  # Production Environment Service
  # ============================
  th-noti-prod:
    build:
      context: .
      args:
        PORT: 3005
    container_name: th-noti-prod
    ports:
      - "3005:3005"
    deploy:
      resources:
        limits:
          cpus: '0.25'        # Maximum 0.25 CPU core
          memory: 256M      # Maximum 256MB RAM
    environment:
      - NODE_ENV=production
      # NextAuth Configuration (REQUIRED)
      - NEXTAUTH_URL=${PROD_NEXTAUTH_URL}
      - NEXTAUTH_SECRET=${NEXTAUTH_SECRET}
      # Database Configuration (REQUIRED)
      - DATABASE_URL=${HOST_PROD_DATABASE_URL}
      # Google OAuth (if using)
      - GOOGLE_CLIENT_ID=${GOOGLE_CLIENT_ID:-}
      - GOOGLE_CLIENT_SECRET=${GOOGLE_CLIENT_SECRET:-}
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:3005/api/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s


  # ============================
  # Test Environment Service
  # ============================
  th-noti-test:
    build:
      context: .
      args:
        PORT: 3006
    container_name: th-noti-test
    ports:
      - "3006:3006"
    deploy:
      resources:
        limits:
          cpus: '0.25'        # Maximum 0.5 CPU core
          memory: 256M      # Maximum 256MB RAM
    environment:
      - NODE_ENV=production
      # NextAuth Configuration (REQUIRED)
      - NEXTAUTH_URL=${TEST_NEXTAUTH_URL}
      - NEXTAUTH_SECRET=${NEXTAUTH_SECRET}
      # Database Configuration (REQUIRED)
      - DATABASE_URL=${HOST_TEST_DATABASE_URL}
      # Google OAuth (if using)
      - GOOGLE_CLIENT_ID=${GOOGLE_CLIENT_ID:-}
      - GOOGLE_CLIENT_SECRET=${GOOGLE_CLIENT_SECRET:-}
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:3006/api/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  
```
## 3. Deployment Steps
### 3.1 Build and Run: Use Docker Compose to build the images and start the services in detached mode (-d).
```
  docker compose up -d --build
```
### 3.2 Verification: Check the status of your running containers:
```
  docker compose ps
```
### 3.3 Stop the Container
```
  sudo docker stop <container_name>
```
### 3.4 Remove the Container
```
  sudo docker rm <container_name>
```
