
# My Open AI Service - Docker Setup

This repository contains a `docker-compose.yml` configuration to set up a local environment for AI services using tools such as Traefik, Ollama, Open WebUI, and Cloudflare's `cloudflared`.

## Prerequisites

Before starting, ensure the following are installed on your system:
- Docker
- Docker Compose

If you plan to use GPU resources for AI processing, ensure you have:
- NVIDIA drivers installed and configured with [NVIDIA Docker toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html#docker).

## Service Overview

This setup provides multiple services that work together to facilitate secure, scalable AI model deployment and management.

### 1. **Cloudflare Tunnel (`cloudflared_tunnel`)**

- **Purpose**: This service sets up a secure tunnel between Cloudflare and your local environment, allowing access to your internal services from the outside world without directly exposing them.
- **Usage**: You need to provide a valid Cloudflare Tunnel Token (`TUNNEL_TOKEN`) in the environment variables to authenticate and run the tunnel.
- **Command**: The service runs the tunnel using the `cloudflared` image with the `--no-autoupdate` flag to avoid auto-updating the tunnel client during runtime.

  **Run without Docker Compose**:
  > `docker run -itd --name cloudflared cloudflare/cloudflared:latest tunnel --no-autoupdate run --token <CLOUDFLARE_TUNNEL_TOKEN>`


### 2. **Traefik (`traefik`)**

- **Purpose**: Traefik is used as a reverse proxy and load balancer to manage requests to your services. It can handle HTTP/HTTPS traffic and automatically acquire SSL certificates using Let's Encrypt (if configured).
- **Configuration**:
  - **Logging**: Set to `TRACE` or `DEBUG` for detailed logging (`--log.level=TRACE` or `--log.level=DEBUG`).
  - **HTTP & HTTPS**: Listens on ports 80 (HTTP) and 443 (HTTPS).
  - **SSL Redirection**: Automatically redirects HTTP to HTTPS. You can disable this by commenting out the related configuration lines in the `docker-compose.yml`.
  - **Optional Let's Encrypt Integration**: There's a commented-out section that, when enabled, allows Traefik to automatically obtain SSL certificates using Cloudflare's DNS challenge.
  - **Comment in the Code**: Comment out marked lines to connect without https / SSL

  This line indicates you can disable HTTPS redirection if SSL is not required.

  **Ports**:
  - **80:80** – HTTP port
  - **443:443** – HTTPS port
  - **9999:8080** – Traefik dashboard exposed on port 9999. => [http://localhost:9999](http://localhost:9999)

### 3. **Whoami (`whoami`)**

- **Purpose**: A simple test service that responds with basic information about the request, such as the hostname and the IP address. This is useful for verifying that Traefik routing and other services are set up correctly.
- **Traefik Labels**:
  - It is exposed via Traefik and can be accessed using a hostname matching the specified domain patterns (`LOCAL_DOMAIN_PATTERN` and `PUBLIC_DOMAIN_PATTERN`).
  - It is set up with TLS (HTTPS) via Cloudflare’s certificate resolver.

  **Labels**:
  - `"traefik.http.routers.whoami.rule=HostRegexp(^whoami\.(${LOCAL_DOMAIN_PATTERN}|${PUBLIC_DOMAIN_PATTERN})$)"` specifies the hostname pattern for accessing the service.

### 4. **Ollama Engine (`ollama_engine`)**

- **Purpose**: Ollama is an AI engine that runs models locally. It supports GPU acceleration for faster model inference and can work with Intel NPUs.
- **Configuration**:
  - **Port**: Exposed on `11434` for external access.
  - **GPU and Intel NPU Support**: You can enable GPU support by uncommenting the related `nvidia` driver configuration, and Intel NPU support by uncommenting the Intel paths and device configuration.
  - **Volumes**: The configuration mounts a local directory for persistent data storage for Ollama models.
  - **Download Models**:  
  Reference to the models: [Click here to find models](https://ollama.com/library)  
  Connect to Ollama container and run commands to pull new models:
    ```bash
    docker exec -it ollama /bin/bash
    > ollama pull llama3.2
    > ollama pull gemma2:2b
    > ollama pull gemma2
    ```


### 5. **Open WebUI (`open_webui`)**

- **Purpose**: A web-based user interface for managing AI models and interacting with them through HTTP requests.
- **Traefik Integration**:
  - Like the `whoami` service, it is exposed via Traefik and secured with HTTPS (optional). You can access the service by visiting a hostname that matches the specified patterns (`HOST_NAME_PATTERN`, `LOCAL_DOMAIN_PATTERN`, or `PUBLIC_DOMAIN_PATTERN`).
  - It connects with Ollama through its API (`OLLAMA_BASE_URL`).

  **Labels**:
  - `"traefik.http.routers.open-webui.rule=HostRegexp(^${HOST_NAME_PATTERN}\.(${LOCAL_DOMAIN_PATTERN}|${PUBLIC_DOMAIN_PATTERN})$)"` specifies the hostname pattern for accessing the service.

## Setup Instructions

### 1. Clone the repository

```bash
git clone <repository-url>
cd <repository-directory>
```

### 2. Configure Environment Variables

Create a `.env` file in the root directory and define the following variables:

```bash
CLOUDFLARE_TUNNEL_TOKEN=<your-cloudflare-tunnel-token>
LOCAL_DOMAIN_PATTERN=<your-local-domain-pattern>
PUBLIC_DOMAIN_PATTERN=<your-public-domain-pattern>
HOST_NAME_PATTERN=<your-host-name-pattern>
```

If you're using Let's Encrypt with Cloudflare for automatic SSL certificates, also define:
```bash
CLOUDFLARE_EMAIL=<your-cloudflare-email>
CLOUDFLARE_API_KEY=<your-cloudflare-api-key>
```

### 3. Start the Services

To start all the services, run:

```bash
sudo docker compose -f network-compose.yml up -d
sudo docker compose up -d
```

- `-d` flag runs the containers in detached mode (in the background).
- This will:
  - Start the Cloudflare Tunnel to securely expose your services.
  - Launch the reverse proxy (Traefik) with SSL redirection and routing rules.
  - Start Ollama, Open WebUI, and the Whoami test service.

### 4. Stop the Services

To stop the running containers:

```bash
sudo docker compose down
sudo docker compose -f network-compose.yml down
```

This stops and removes the containers but keeps the volumes and network configurations.
To remove the volumes, use the command:
```bash
sudo docker compose down -v
sudo docker compose -f network-compose.yml down -v
```
The content of the volumes which have specific local path defined, will not be removed. The path can be remounted at the next run.

## Troubleshooting

- **Cloudflare Tunnel not connecting**: Ensure your `TUNNEL_TOKEN` is correct and that your Cloudflare account is properly set up.
- **SSL Issues**: If Let's Encrypt SSL certificates are not being issued, verify your Cloudflare API credentials and ensure DNS settings for your domain are correctly configured.

## Optional Configurations

- **Disabling HTTPS/SSL**: If you don't want to use HTTPS, comment out the relevant SSL-related lines in the Traefik configuration.
- **GPU/Intel NPU**: Ensure that hardware support is correctly configured on your host system. If you're not using GPUs or Intel NPUs, you can comment out those sections in the `docker-compose.yml`.

## Useful Resources

- [Traefik Documentation](https://doc.traefik.io/traefik/)
- [Cloudflare Tunnel Documentation](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/)
- [Ollama Engine Docker Setup](https://ollama.com/blog/ollama-is-now-available-as-an-official-docker-image)
