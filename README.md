# Open Web UI - AI Stack with Docker Compose v2 & NVIDIA GPU

### Build your private and safe  generative artificial intelligence chatbot with a modern UI including image quering, real-time web search using latest open source Large Language Models such as llama 3.2 vision or llama 3.3 and searXNG.

This repository provides a full-stack UI solution using Docker Compose, integrating various AI tools and services, including Open Web UI, Ollama, SearXNG, Redis, and Caddy. This setup is designed to leverage NVIDIA GPU acceleration for running AI models. But also works with CPU - although performances are poor

## Features

- **Open Web UI**: A user-friendly interface for interacting with various AI models.
- **Ollama**: A service to serve and interact with large language models (LLMs).
- **SearXNG**: A privacy-respecting search engine for querying multiple sources.
- **Redis**: Lightweight in-memory data store used for caching.
- **Caddy**: A reverse proxy server for routing traffic and handling SSL/TLS (if required).
- **GPU Acceleration**: NVIDIA GPU support for containers running on compatible systems.

## Prerequisites

Before running the project, ensure the following dependencies are installed on your machine:

1. **Docker**: The containerization platform to run all services.
   - [Docker Installation Guide](https://docs.docker.com/get-docker/)
2. **NVIDIA Docker**: To enable GPU support for Docker containers.

   - [Install NVIDIA Docker](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html)

3. **NVIDIA Runtime**: Ensure your Docker runtime is set to `nvidia` for GPU acceleration.

   - Ensure the default runtime is configured in `/etc/docker/daemon.json`:
     ```json
     {
       "runtimes": {
         "nvidia": {
           "path": "/usr/bin/nvidia-container-runtime",
           "runtimeArgs": []
         }
       },
       "default-runtime": "nvidia"
     }
     ```

4. **Docker Compose v2**: A tool to define and manage multi-container Docker applications.

   - [Docker Compose Installation Guide](https://docs.docker.com/compose/install/)

### AI stack

- **Open Web UI** : https://github.com/open-webui/open-webui

- **Ollama** : https://ollama.com/

- **Searxng** : https://github.com/searxng/searxng

# Project Setup

## 1. **Clone the Repository**

```bash
  git clone https://github.com/maximesoydas/OpenWebUi-AI-Stack.git
  cd OpenWebUi-AI-Stack
```

## 2. **Configure Docker Compose File**

Ensure the docker-compose.yml file is set up as follows. This file defines the services, networking, volumes, and GPU configurations for all your containers

```yaml
version: "3.8" # Specify the Compose file format version. Version 3.8 ensures compatibility with Docker Compose v2.

services: # Define all the services to be orchestrated by Docker Compose.
  open-webui: # Define the Open Web UI service.
    image: ghcr.io/open-webui/open-webui:main # Use the main image from the GitHub container registry.
    container_name: open-webui # Assign a specific name to the container.
    ports: # Map the container's internal port 8080 to the host's port 3333.
      - "3333:8080"
    environment: # Define environment variables for Open Web UI.
      OLLAMA_BASE_URL: "http://ollama:11434" # Point Open Web UI to the Ollama service.
      SEARXNG_BASE_URL: "http://searxng:8080" # Point Open Web UI to the SearXNG service.
    volumes: # Persist data in a named volume.
      - open-webui-data:/app/backend/data
    depends_on: # Ensure Open Web UI starts after SearXNG and Ollama.
      - searxng
      - ollama
    restart: always # Automatically restart the container if it stops or crashes.
    # NOTE: Docker Compose v2 does not fully support `--gpus` yet; use `deploy` instead.
    deploy: # Deployment settings for resource reservations.
      resources:
        reservations:
          devices: # Reserve all GPUs for this service.
            - driver: nvidia
              count: all
              capabilities: [gpu]

  ollama: # Define the Ollama service.
    image: ollama/ollama:latest # Use the latest Ollama image.
    container_name: ollama # Assign a specific name to the container.
    ports: # Map the container's internal port 11434 to the host's port 11434.
      - "11434:11434"
    command: serve # Run the "serve" command as the container's entry point.
    volumes: # Persist data in a named volume for Ollama.
      - ollama-data:/root/.ollama
    restart: always # Automatically restart the container if it stops or crashes.

  searxng: # Define the SearXNG service.
    image: searxng/searxng:latest # Use the latest SearXNG image.
    container_name: searxng # Assign a specific name to the container.
    ports: # Map the container's internal port 8080 to the host's port 5555.
      - "5555:8080"
    environment: # Define environment variables for SearXNG.
      SEARXNG_SETTINGS_PATH: "/etc/searxng/settings.yml" # Path to the settings configuration file.
    volumes: # Mount settings and other configuration files into the container.
      - ./searxng/settings.yml:/etc/searxng/settings.yml # Mount a specific configuration file.
      - ./searxng:/etc/searxng # Alternatively, mount the entire directory for flexibility.
    restart: always # Automatically restart the container if it stops or crashes.

  redis: # Define the Redis service.
    image: redis:alpine # Use the lightweight Alpine Redis image.
    container_name: redis # Assign a specific name to the container.
    restart: always # Automatically restart the container if it stops or crashes.
    ports: # Map the container's internal port 6379 to the host's port 6379.
      - "6379:6379"

  caddy: # Define the Caddy service (reverse proxy).
    image: caddy:latest # Use the latest Caddy image.
    container_name: caddy # Assign a specific name to the container.
    ports: # Expose ports for HTTP (80) and HTTPS (443).
      - "80:80"
      - "443:443"
    volumes: # Mount configuration and data volumes.
      - ./caddy/Caddyfile:/etc/caddy/Caddyfile # Mount the Caddyfile for configuration.
      - caddy_data:/data # Persist Caddy's runtime data.
      - caddy_config:/config # Persist Caddy's configuration data.
    depends_on: # Ensure Caddy starts after dependent services.
      - open-webui
      - ollama
      - searxng
    restart: always # Automatically restart the container if it stops or crashes.

volumes: # Define named volumes for persistent storage.
  open-webui-data: # Volume for Open Web UI data.
  ollama-data: # Volume for Ollama data.
  caddy_data: # Volume for Caddy runtime data.
  caddy_config: # Volume for Caddy configuration data.
```

## 3. **SearXNG UWSGI Configuration (uwsgi.ini)**

For serving the searxng app, ensure the UWSGI configuration file (uwsgi.ini) is set up with the following:

```ini
[uwsgi]
uid = searxng  # User ID under which the uWSGI process runs, ensuring proper permissions.
gid = searxng  # Group ID for the uWSGI process, matching the user group.

workers = %k  # Number of worker processes, typically set to the number of CPU cores.
threads = 4  # Number of threads per worker process for handling multiple requests simultaneously.

plugin = python3  # Specifies the Python 3 plugin for uWSGI to use Python applications.
master = true  # Enables master process mode, improving resource management and restarting capabilities.

module = searx.webapp  # Defines the WSGI entry point module for SearXNG.

pythonpath = /usr/local/searxng/  # Path to the Python environment where SearXNG is installed.
chdir = /usr/local/searxng/searx/  # Working directory for uWSGI, pointing to the SearXNG application.

static-map = /static=/usr/local/searxng/searx/static  # Serves static files from the specified directory.
static-expires = /* 86400  # Sets a cache expiration for static files (in seconds), here one day (86400 seconds).
static-gzip-all = True  # Enables gzip compression for all served static files to optimize bandwidth.

```

## 4. **SearXNG Settings (settings.yml)**

The SearXNG settings are configured to your requirements, including setting use_default_settings: true.

```bash
#Generate the secret key with following command in your terminal and copy it in secret_key below 'in strings'

openssl rand -hex 32
```

```yaml
use_default_settings: true
server:
  secret_key: "the result from the command mentioned right above"
limiter: false
image_proxy: true
port: 8080
bind_address: "0.0.0.0"
```

## 5. **Limiter settings (limiter.toml)**

```toml
[botdetection.ip_limit]
# activate link_token method in the ip_limit method
link_token = true
```

## 5. **Caddyfile**

The Caddyfile configures reverse proxies for each service (Open Web UI, SearXNG, Ollama)

```caddy
localhost:3333 {
  reverse_proxy 127.0.0.1:8080
}

localhost:5555 {
  reverse_proxy 127.0.0.1:8080
}

localhost:11434 {
  reverse_proxy 127.0.0.1:11434
}
```

## 6. **Running the Stack**

Works best with docker compose v2

- Run the service :
```bash
docker compose up -d
```

- Stop the service:
```bash
docker compose down
```
- Show the containers:
```bash
docker psa -a
```

### Access the Services

- Open Web UI: http://localhost:3333

- SearXNG: http://localhost:5555

- Ollama: http://localhost:11434

### Download LLM Model

Now you will need to download the model you want, i suggest [llama3.2-vision](https://ollama.com/library/llama3.2-vision) you can also check the whole models list at [List All Models](https://ollama.com/search)

```bash
docker exec -it ollama /bin/bash
# for this project :
ollama run llama3.2-vision
```

#### [List All Models](https://ollama.com/search)

```bash
ollama run <model-name>
```

## 7. Open Web UI Configuration (Connecting Models, Web Search)

Go to http://localhost:3333/admin/settings

Connecting Model with local OLLAMA API :

- user profile picture -> Admin Panel -> Settings (tab) -> **Connections**
- Enable Ollama API
- Click and on +
- URL: http://ollama:11434
- No need for a prefix nor for an api key leave these blank.

Connecting Web Search with local SearXNG Query URL :

- User profile picture -> Admin Panel -> Settings (tab) -> **Web Search**
- Enable Web Search
- Web Search Engine : searxng
- Searxng Query URL : `http://searxng:8080/search?q=<query>`

Feel free to restart the docker compose containers whenever :

- Stop and Remove ALL docker containers

```bash
docker stop $(docker ps -aq) && docker rm $(docker ps -aq)
```

- Stop the service

```bash
docker compose down
```

- Run the service

```bash
docker compose up -d
```

- Show the containers

```bash
docker psa -a
```

## 8. Troubleshooting

- Logs: Check service logs to debug issues:

```bash
docker compose logs <service-name>
```

GPU Issues: Verify NVIDIA Toolkit installation and runtime configuration.

You can also remove the nvidia parts and gpu part in the docker-compose.yaml for CPU use only, although it will result in slower performances depending on the model used.

## Contribution

We welcome contributions! Feel free to open issues, submit pull requests, or share feedback.

## Licence

This `README.md` incorporates Docker Compose v2 nuances and NVIDIA GPU support, providing step-by-step setup instructions. Let me know if there are further tweaks you'd like!
