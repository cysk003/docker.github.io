---
title: Containerize a React.js Application
linkTitle: Containerize
weight: 10
keywords: react.js, node, image, initialize, build
description: Learn how to containerize a React.js application with Docker by creating an optimized, production-ready image using best practices for performance, security, and scalability.

---

## Prerequisites

Before you begin, make sure the following tools are installed and available on your system:

- You have installed the latest version of [Docker Desktop](/get-started/get-docker.md).
- You have a [git client](https://git-scm.com/downloads). The examples in this section use a command-line based git client, but you can use any client.

> **New to Docker?**  
> Start with the [Docker basics](/get-started/docker-concepts/the-basics/what-is-a-container.md) guide to get familiar with key concepts like images, containers, and Dockerfiles.

---

## Overview

This guide walks you through the complete process of containerizing a React.js application with Docker. You’ll learn how to create a production-ready Docker image using best practices that improve performance, security, scalability, and deployment efficiency.

By the end of this guide, you will:

- Containerize a React.js application using Docker.
- Create and optimize a Dockerfile for production builds. 
- Use multi-stage builds to minimize image size.
- Serve the application efficiently with a custom NGINX configuration.
- Follow best practices for building secure and maintainable Docker images. 

---

## Get the sample application

Clone the sample application to use with this guide. Open a terminal, change
directory to a directory that you want to work in, and run the following command
to clone the git repository:

```console
$ git clone https://github.com/kristiyan-velkov/docker-reactjs-sample
```
---

## Generate a Dockerfile

Docker provides an interactive CLI tool called `docker init` that helps scaffold the necessary configuration files for containerizing your application. This includes generating a `Dockerfile`, `.dockerignore`, `compose.yaml`, and `README.Docker.md`.

To begin, navigate to the root of your project directory:

```console
$ cd docker-reactjs-sample
```

Then run the following command:

```console
$ docker init
```
You’ll see output similar to:

```text
Welcome to the Docker Init CLI!

This utility will walk you through creating the following files with sensible defaults for your project:
  - .dockerignore
  - Dockerfile
  - compose.yaml
  - README.Docker.md

Let's get started!
```

The CLI will prompt you with a few questions about your app setup.
For consistency, please use the same responses shown in the example below when prompted:
| Question                                                   | Answer          |
|------------------------------------------------------------|-----------------|
| What application platform does your project use?           | Node            |
| What version of Node do you want to use?                   | 22.14.0-alpine  |
| Which package manager do you want to use?                  | npm             |
| Do you want to run "npm run build" before starting server? | yes             |
| What directory is your build output to?                    | dist            |
| What command do you want to use to start the app?          | npm run dev     |
| What port does your server listen on?                      | 8080            |

After completion, your project directory will contain the following new files:

```text
├── docker-reactjs-sample/
│ ├── Dockerfile
│ ├── .dockerignore
│ ├── compose.yaml
│ └── README.Docker.md
```

---

## Build the Docker image

The default Dockerfile generated by `docker init` serves as a solid starting point for general Node.js applications. However, React.js is a front-end library that compiles into static assets, so we need to tailor the Dockerfile to optimize for how React applications are built and served in a production environment.

### Step 1: Review the generated files

In this step, you’ll improve the Dockerfile and configuration files by following best practices:

- Use multi-stage builds to keep the final image clean and small  
- Serve the app using NGINX, a fast and secure web server  
- Improve performance and security by only including what’s needed  

These updates help ensure your app is easy to deploy, fast to load, and production-ready.

> [!NOTE]
> A `Dockerfile` is a plain text file that contains step-by-step instructions to build a Docker image. It automates packaging your application along with its dependencies and runtime environment.  
> For full details, see the [Dockerfile reference](/reference/dockerfile/).


### Step 2: Configure the Dockerfile file 

Copy and replace the contents of your existing `Dockerfile` with the configuration below:

```dockerfile
# =========================================
# Stage 1: Build the React.js Application
# =========================================
ARG NODE_VERSION=22.14.0-alpine
ARG NGINX_VERSION=alpine3.21

# Use a lightweight Node.js image for building (customizable via ARG)
FROM node:${NODE_VERSION} AS builder

# Set the working directory inside the container
WORKDIR /app

# Copy package-related files first to leverage Docker's caching mechanism
COPY package.json package-lock.json ./

# Install project dependencies using npm ci (ensures a clean, reproducible install)
RUN --mount=type=cache,target=/root/.npm npm ci

# Copy the rest of the application source code into the container
COPY . .

# Build the React.js application (outputs to /app/dist)
RUN npm run build

# =========================================
# Stage 2: Prepare Nginx to Serve Static Files
# =========================================

FROM nginxinc/nginx-unprivileged:${NGINX_VERSION} AS runner

# Use a built-in non-root user for security best practices
USER nginx

# Copy custom Nginx config
COPY nginx.conf /etc/nginx/nginx.conf

# Copy the static build output from the build stage to Nginx's default HTML serving directory
COPY --chown=nginx:nginx  --from=builder /app/dist /usr/share/nginx/html

# Expose port 8080 to allow HTTP traffic
# Note: The default NGINX container now listens on port 8080 instead of 80 
EXPOSE 8080

# Start Nginx directly with custom config
ENTRYPOINT ["nginx", "-c", "/etc/nginx/nginx.conf"]
CMD ["-g", "daemon off;"]
```

### Step 3: Configure the .dockerignore file

The `.dockerignore` file tells Docker which files and folders to exclude when building the image.


> [!NOTE]
>This helps:
>- Reduce image size  
>- Speed up the build process  
>- Prevent sensitive or unnecessary files (like `.env`, `.git`, or `node_modules`) from being added to the final image.
>
> To learn more, visit the [.dockerignore reference](/reference/dockerfile.md#dockerignore-file).

Copy and replace the contents of your existing `.dockerignore` with the configuration below:

```dockerignore
# Ignore dependencies and build output
node_modules/
dist/
out/
.tmp/
.cache/

# Ignore Vite, Webpack, and React-specific build artifacts
.vite/
.vitepress/
.eslintcache
.npm/
coverage/
jest/
cypress/
cypress/screenshots/
cypress/videos/
reports/

# Ignore environment and config files (sensitive data)
*.env*
*.log

# Ignore TypeScript build artifacts (if using TypeScript)
*.tsbuildinfo

# Ignore lockfiles (optional if using Docker for package installation)
npm-debug.log*
yarn-debug.log*
yarn-error.log*
pnpm-debug.log*

# Ignore local development files
.git/
.gitignore
.vscode/
.idea/
*.swp
.DS_Store
Thumbs.db

# Ignore Docker-related files (to avoid copying unnecessary configs)
Dockerfile
.dockerignore
docker-compose.yml
docker-compose.override.yml

# Ignore build-specific cache files
*.lock

```

### Step 4: Create the `nginx.conf` file

To serve your React.js application efficiently inside the container, you’ll configure NGINX with a custom setup. This configuration is optimized for performance, browser caching, gzip compression, and support for client-side routing.

Create a file named `nginx.conf` in the root of your project directory, and add the following content:

> [!NOTE]
> To learn more about configuring NGINX, see the [official NGINX documentation](https://nginx.org/en/docs/).


```nginx
worker_processes auto;

# Store PID in /tmp (always writable)
pid /tmp/nginx.pid;

events {
    worker_connections 1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    # Disable logging to avoid permission issues
    access_log off;
    error_log  /dev/stderr warn;

    # Optimize static file serving
    sendfile        on;
    tcp_nopush      on;
    tcp_nodelay     on;
    keepalive_timeout  65;
    keepalive_requests 1000;

    # Gzip compression for optimized delivery
    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript image/svg+xml;
    gzip_min_length 256;
    gzip_vary on;

    server {
        listen       8080;
        server_name  localhost;

        # Root directory where React.js build files are placed
        root /usr/share/nginx/html;
        index index.html;

        # Serve React.js static files with proper caching
        location / {
            try_files $uri /index.html;
        }

        # Serve static assets with long cache expiration
        location ~* \.(?:ico|css|js|gif|jpe?g|png|woff2?|eot|ttf|svg|map)$ {
            expires 1y;
            access_log off;
            add_header Cache-Control "public, immutable";
        }

        # Handle React.js client-side routing
        location /static/ {
            expires 1y;
            add_header Cache-Control "public, immutable";
        }
    }
}
```

### Step 5: Build the React.js application image

With your custom configuration in place, you're now ready to build the Docker image for your React.js application.

The updated setup includes:

- Optimized browser caching and gzip compression  
- Secure, non-root logging to avoid permission issues  
- Support for React client-side routing by redirecting unmatched routes to `index.html`  

After completing the previous steps, your project directory should now contain the following files:

```text
├── docker-reactjs-sample/
│ ├── Dockerfile
│ ├── .dockerignore
│ ├── compose.yaml
│ ├── nginx.conf
│ └── README.Docker.md
```

Now that your Dockerfile is configured, you can build the Docker image for your React.js application.

> [!NOTE]
> The `docker build` command packages your application into an image using the instructions in the Dockerfile. It includes all necessary files from the current directory (called the [build context](/build/concepts/context/#what-is-a-build-context)).

Run the following command from the root of your project:

```console
$ docker build --tag docker-reactjs-sample .
```

What this command does:
- Uses the Dockerfile in the current directory (.)
- Packages the application and its dependencies into a Docker image
- Tags the image as docker-reactjs-sample so you can reference it later


#### Step 6:  View local images

After building your Docker image, you can check which images are available on your local machine using either the Docker CLI or [Docker Desktop](/manuals/desktop/use-desktop/images.md). Since you're already working in the terminal, let's use the Docker CLI.

To list all locally available Docker images, run the following command:

```console
$ docker images
```

Example Output:

```shell
REPOSITORY                TAG               IMAGE ID       CREATED         SIZE
docker-reactjs-sample     latest            f39b47a97156   14 seconds ago   75.8MB
```

This output provides key details about your images:

- **Repository** – The name assigned to the image.
- **Tag** – A version label that helps identify different builds (e.g., latest).
- **Image ID** – A unique identifier for the image.
- **Created** – The timestamp indicating when the image was built.
- **Size** – The total disk space used by the image.

If the build was successful, you should see `docker-reactjs-sample` image listed. 

---

## Run the containerized application

In the previous step, you created a Dockerfile for your React.js application and built a Docker image using the docker build command. Now it’s time to run that image in a container and verify that your application works as expected.


Inside the `docker-reactjs-sample` directory, run the following command in a
terminal.

```console
$ docker compose up --build
```

Open a browser and view the application at [http://localhost:8080](http://localhost:8080). You should see a simple React.js web application.

Press `ctrl+c` in the terminal to stop your application.

### Run the application in the background

You can run the application detached from the terminal by adding the `-d`
option. Inside the `docker-reactjs-sample` directory, run the following command
in a terminal.

```console
$ docker compose up --build -d
```

Open a browser and view the application at [http://localhost:8080](http://localhost:8080). You should see a simple web application preview.


To confirm that the container is running, use `docker ps` command:

```console
$ docker ps
```

This will list all active containers along with their ports, names, and status. Look for a container exposing port 8080.

Example Output:

```shell
CONTAINER ID   IMAGE                          COMMAND                  CREATED             STATUS             PORTS                    NAMES
88bced6ade95   docker-reactjs-sample-server   "nginx -c /etc/nginx…"   About a minute ago  Up About a minute  0.0.0.0:8080->8080/tcp   docker-reactjs-sample-server-1
```


To stop the application, run:

```console
$ docker compose down
```


> [!NOTE]
> For more information about Compose commands, see the [Compose CLI
> reference](/reference/cli/docker/compose/_index.md).

---

## Summary

In this guide, you learned how to containerize, build, and run a React.js application using Docker. By following best practices, you created a secure, optimized, and production-ready setup.

What you accomplished:
- Initialized your project using `docker init` to scaffold essential Docker configuration files.
- Replaced the default `Dockerfile` with a multi-stage build that compiles the React.js application and serves the static files using Nginx.
- Replaced the default `.dockerignore` file to exclude unnecessary files and keep the image clean and efficient.
- Built your Docker image using `docker build`.
- Ran the container using `docker compose up`, both in the foreground and in detached mode.
- Verified that the app was running by visiting [http://localhost:8080](http://localhost:8080).
- Learned how to stop the containerized application using `docker compose down`.

You now have a fully containerized React.js application, running in a Docker container, and ready for deployment across any environment with confidence and consistency.

---

## Related resources

Explore official references and best practices to sharpen your Docker workflow:

- [Multi-stage builds](/build/building/multi-stage/) – Learn how to separate build and runtime stages.
- [Best practices for writing Dockerfiles](/develop/develop-images/dockerfile_best-practices/) – Write efficient, maintainable, and secure Dockerfiles.  
- [Build context in Docker](/build/concepts/context/) – Learn how context affects image builds.  
- [`docker init` CLI reference](/reference/cli/docker/init/) – Scaffold Docker assets automatically.
- [`docker build` CLI reference](/reference/cli/docker/build/) – Build Docker images from a Dockerfile.
- [`docker images` CLI reference](/reference/cli/docker/images/) – Manage and inspect local Docker images.
- [`docker compose up` CLI reference](/reference/cli/docker/compose/up/) – Start and run multi-container applications.
- [`docker compose down` CLI reference](/reference/cli/docker/compose/down/) – Stop and remove containers, networks, and volumes.

---

## Next steps

With your React.js application now containerized, you're ready to move on to the next step.

In the next section, you'll learn how to develop your application using Docker containers, enabling a consistent, isolated, and reproducible development environment across any machine.

