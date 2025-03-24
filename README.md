# Docker First Visit

# Creating a Dockerized Web Application Tutorial for Full-Stack Development

This tutorial will guide you through the process of dockerizing a full-stack web application, following the curriculum of DTI 201 Full-Stack Software Development Skills. By the end of this tutorial, you'll understand how to containerize both frontend and backend components of your application.

## Prerequisites

Before starting, make sure you have:
- Docker installed on your computer
- Basic understanding of web development concepts
- Your full-stack project with frontend (React) and backend (Node.js)

## Part 1: Understanding Docker Basics

### What is Docker?

Docker is a platform that allows you to develop, ship, and run applications in containers. Containers are lightweight, portable, and consistent environments that package your application along with its dependencies.

### Key Docker Concepts

- **Dockerfile**: A text file containing instructions to build a Docker image
- **Image**: A template for creating containers
- **Container**: A running instance of an image
- **Docker Compose**: A tool for defining and running multi-container applications

### Essential Docker Commands

1. `docker ps` - List running containers
   - Format output: `docker ps --format=''`
   - Use environment variables: `docker ps --format=$Env:FORMAT`

2. `docker images` - List all images on your machine

3. `docker run -ti ubuntu:latest bash` - Run a container from an image
   - `-ti` provides an interactive terminal
   - `--rm` automatically removes the container when it exits

4. `docker commit <ID>` - Create a new image from a container's changes

5. `docker tag <ImageID> <tag>` - Assign a name to an image

## Part 2: Hands-on Activity: Building a Node.js Server Container

### Step 1: Create a Basic Ubuntu Container with Node.js

```bash
# Run an Ubuntu container with an interactive terminal
docker run --rm -ti ubuntu bash

# Inside the container, update package lists
apt-get update

# Install necessary tools
apt-get install -y netcat
apt-get install -y nodejs
apt-get install -y npm

# Verify installations
node --version
npm --version
```

### Step 2: Create a Simple Echo Server using Netcat

Once you have your container running with the necessary tools, we'll set up a simple echo server:

1. Save the container as a new image:
   ```bash
   # Exit the container
   exit
   
   # Find the container ID
   docker ps -a
   
   # Commit the changes to a new image
   docker commit <CONTAINER_ID> nodejs_server
   ```

2. Run a new container with port mapping:
   ```bash
   docker run --rm -ti -p 45678:45678 -p 45679:45679 --name echo-server nodejs_server bash
   ```

3. Inside the container, start a netcat echo server:
   ```bash
   nc -lp 45678 | nc -lp 45679
   ```

4. From your host machine, test the connection:
   ```bash
   # For Windows
   nc <your-ip-address> 45678
   
   # For Mac
   nc host.docker.internal 45678
   ```

Any message sent to port 45678 will be echoed back on port 45679.

## Part 3: Dockerizing the Backend (Node.js/Express)

### Step 1: Create a Dockerfile for your Backend

Create a file named `Dockerfile` in your backend directory with the following content:

```dockerfile
# Use Node.js LTS version
FROM node:18-alpine

# Set working directory
WORKDIR /app

# Copy package.json and package-lock.json
COPY package*.json ./

# Install dependencies
RUN npm install

# Copy remaining source code
COPY . .

# Expose the port your app runs on
EXPOSE 3001

# Command to run the application
CMD ["npm", "start"]
```

### Step 2: Create a .dockerignore File

Create a `.dockerignore` file to exclude unnecessary files:

```
node_modules
npm-debug.log
.git
.env
```

## Part 4: Dockerizing the Frontend (React)

### Step 1: Create a Dockerfile for your Frontend

Create a `Dockerfile` in your frontend directory:

```dockerfile
# Build stage
FROM node:18-alpine as build

# Set working directory
WORKDIR /app

# Copy package.json and package-lock.json
COPY package*.json ./

# Install dependencies
RUN npm install

# Copy source code
COPY . .

# Build the application
RUN npm run build

# Production stage
FROM nginx:alpine

# Copy built files from build stage to nginx directory
COPY --from=build /app/dist /usr/share/nginx/html

# Copy nginx configuration if you have custom settings
# COPY nginx.conf /etc/nginx/conf.d/default.conf

# Expose port 80
EXPOSE 80

# Start nginx
CMD ["nginx", "-g", "daemon off;"]
```

### Step 2: Create a .dockerignore File for Frontend

```
node_modules
npm-debug.log
.git
dist
```

## Part 5: Setting Up Docker Compose

Create a `docker-compose.yml` file in your project root directory:

```yaml
version: '3.8'

services:
  # Backend service
  backend:
    build: ./backend
    container_name: fullstack-backend
    ports:
      - "3001:3001"
    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgres://postgres:password@db:5432/fullstack_db
    depends_on:
      - db
    networks:
      - fullstack-network

  # Frontend service
  frontend:
    build: ./frontend
    container_name: fullstack-frontend
    ports:
      - "80:80"
    depends_on:
      - backend
    networks:
      - fullstack-network

  # Database service
  db:
    image: postgres:14-alpine
    container_name: fullstack-db
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=fullstack_db
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - fullstack-network

networks:
  fullstack-network:
    driver: bridge

volumes:
  postgres_data:
```

## Part 6: Working with Docker Containers

### Understanding Container Lifecycle

1. **Creating and starting containers:**
   ```bash
   # Create and start a container in detached mode
   docker run -d -ti ubuntu bash
   
   # List running containers
   docker ps
   ```

2. **Attaching to running containers:**
   ```bash
   # Attach to a running container
   docker attach <container_id_or_name>
   
   # Execute a command in a running container
   docker exec -it <container_id_or_name> /bin/bash
   ```

3. **Working with stopped containers:**
   ```bash
   # Start a stopped container
   docker start <container_id_or_name>
   
   # Check container logs
   docker logs <container_id_or_name>
   ```

### Container Networking Examples

1. **Port mapping for multiple services:**
   ```bash
   # Run a container with multiple port mappings
   docker run --rm -ti -p 45678:45678 -p 45679:45679 --name multi-port-server ubuntu bash
   ```

2. **Testing inter-container communication:**
   - From container 1:
     ```bash
     nc -lp 45678 | nc -lp 45679
     ```
   - From your host machine:
     ```bash
     # For Windows
     nc <your-ip-address> 45678
     
     # For Mac
     nc host.docker.internal 45678
     ```

## Part 7: Running Your Dockerized Application

### Step 1: Build and Start the Containers

Run the following command in your project root directory:

```bash
docker-compose up --build
```

This command will:
1. Build the Docker images for your frontend and backend
2. Create and start the containers
3. Set up the network between containers

### Step 2: Accessing Your Application

- Frontend: http://localhost
- Backend API: http://localhost:3001
- Database: PostgreSQL running on port 5432

## Part 8: Handling Environment Variables

### Creating Environment Files

For development and production environments, create appropriate `.env` files:

**Backend .env.development:**
```
NODE_ENV=development
PORT=3001
DATABASE_URL=postgres://postgres:password@db:5432/fullstack_db
```

**Backend .env.production:**
```
NODE_ENV=production
PORT=3001
DATABASE_URL=postgres://postgres:password@db:5432/fullstack_db
```

### Updating Docker Compose for Environment Variables

```yaml
# Backend service with environment file
backend:
  build: ./backend
  container_name: fullstack-backend
  ports:
    - "3001:3001"
  env_file:
    - ./backend/.env.production
  depends_on:
    - db
  networks:
    - fullstack-network
```

## Part 9: Troubleshooting Docker Containers

### Checking Container Logs

If a container exits unexpectedly:

```bash
# Check the logs to see what happened
docker logs <container_name>

# Example from the notes
docker run --name example -d ubuntu bash -c "lose /etc/passwd"
# This will exit with status code 127 (command not found)
docker logs example
# Output: bash: lose: command not found
```

### Common Issues and Solutions

1. **Command not found errors:**
   - Ensure the required packages are installed in your container
   - Verify paths in your Dockerfile

2. **Connection refused errors:**
   - Check if the container is running: `docker ps`
   - Verify port mappings: `docker port <container_name>`
   - Confirm your application is listening on the correct interface (0.0.0.0)

3. **File permissions issues:**
   - Set appropriate permissions in your Dockerfile
   - Use volumes for persistent data with correct ownership

## Conclusion

Dockerizing your full-stack application provides consistency across development, testing, and production environments. It simplifies deployment and ensures that your application works the same way regardless of where it runs.

By following this tutorial, you've learned how to:
- Create and manage Docker containers
- Build custom Docker images
- Configure networking between containers
- Create Dockerfiles for frontend and backend
- Set up Docker Compose for multi-container orchestration
- Troubleshoot common Docker issues
