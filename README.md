
# Running Open AI Chat locally with Docker under Linux and WSL
**This documentation helps you to set up a Local AI Chat with Ollama and Open-WebUI accessable from the Internet. Base OS can be Windows with WSL or Linux.**  
After a pre-setup, this repository helps to run a Container environment 
This repository contains `network-compose.yml` and `docker-compose.yml` configuration to set up a local environment for AI services using tools such as Traefik, Ollama, Open WebUI, and Cloudflare's `cloudflared`.

## Prerequisites

Before starting, ensure the following are installed on your system:
- [Ubuntu on WSL2 or Linux (Ubuntu)](https://documentation.ubuntu.com/wsl/en/latest/guides/install-ubuntu-wsl2/)
- [Docker](https://docs.docker.com/engine/install/ubuntu/)
- Docker Compose2  
  if not installed with docker install `sudo apt-get install -y docker-compose-v2`
- Cloudflare account and a Domain managed by Cloudflare
- VS Code is recommanded

## Pre-installation
- NVIDIA GPU installation and configuration:  
  Install [NVIDIA Docker toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html#docker)
- [Ollama installation in WSL](https://ollama.com/download/linux)
- Change of Ollama configuration  
  1) Modify Ollama configuration:  
      Edit the Ollama service file to bind it to all network interfaces:
      ```
      sudo nano /etc/systemd/system/ollama.service
      ```
      Add the following line under the [Service] section (Variable Environment must be added a second time if it already exists):  
      ```
      Environment="OLLAMA_HOST=0.0.0.0"
      ```
  2) Restart Ollama:
      ```
      sudo systemctl daemon-reload
      sudo systemctl restart ollama
      ```
  3) Configure firewall (skip in WSL or Linux without Firewall):  
      If you have a firewall enabled, allow incoming connections on port 11434:
      ```
      sudo ufw allow 11434/tcp
      ```
- Download models for Ollama:  
  Reference to the models: [Click here to find models](https://ollama.com/library)  
  Run commands in the bash to pull new models:
    ```bash
    > ollama pull llama3.2
    > ollama pull gemma2:2b
    > ollama pull gemma2
    ```

## Setup Instructions
### 1. Clone the repository
```bash
git clone <repository-url>
cd <repository-directory>
```

### 2. Configure Environment Variables
Create a `.env` file in the root directory and define the following variables:
```bash
LOCAL_DOMAIN_PATTERN="<your-local-domain-pattern>"
PUBLIC_DOMAIN_PATTERN="<your-public-domain-pattern>"
HOST_NAME_PATTERN="<your-host-name-pattern>"

DOCKER_VOLUMES_PATH="./docker_volumes/my-open-ai"
WSL_ETH0_IP="<wsl_eth0_ip>"

CLOUDFLARE_TUNNEL_TOKEN="<your-cloudflare-tunnel-token>"
```
**Enter the IP of WSL eth0 interface or the main linux interface into `WSL_ETH0_IP`**

### 3. Start the Services
To start all the services, run:
```bash
sudo docker compose -f network-compose.yml up -d
sudo docker compose up -d
```
- `-d` flag runs the containers in detached mode (in the background). For debugging it is recommanded to run not in detached mode to get the logs.
- This will:
  - Start the Cloudflare Tunnel to securely expose your services.
  - Launch the reverse proxy (Traefik) with SSL redirection and routing rules.
  - Start Ollama, Open WebUI, and the Whoami test service.

### 4. Stop the Services
This stops and removes the containers but keeps the volumes and network configurations. Remove `-v` to keep Mounts.  
Command:
```bash
sudo docker compose down -v
sudo docker compose -f network-compose.yml down -v
```
The content of the volumes which have specific local path defined, will not be removed. The path can be remounted at the next run.

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
  - **HTTP & HTTPS**: Listens on ports 89 (HTTP) and 4443 (HTTPS).
  - **SSL Redirection**: Automatically redirects HTTP to HTTPS. You can disable this by commenting out the related configuration lines in the `docker-compose.yml`.
  - **Optional Let's Encrypt Integration**: There's a commented-out section that, when enabled, allows Traefik to automatically obtain SSL certificates using Cloudflare's DNS challenge.
  - **Comment in the Code**: Comment out marked lines to connect without https / SSL

  This line indicates you can disable HTTPS redirection if SSL is not required.

  **Ports**:
  - **89:89** – HTTP port
  - **4443:4443** – HTTPS port
  - **9880:8080** – Traefik dashboard exposed on port 9999. => [http://localhost:9880](http://localhost:9880)

### 3. **Open WebUI (`open_webui`)**

- **Purpose**: A web-based user interface for managing AI models and interacting with them through HTTP requests.
- **Traefik Integration**:
  - Open-WebUI service, it is exposed via Traefik and secured with HTTPS (optional). You can access the service by visiting a hostname that matches the specified patterns (`HOST_NAME_PATTERN`, `PUBLIC_DOMAIN_PATTERN`, or `LOCAL_DOMAIN_PATTERN` when tsl is deactivated).
  - It connects with Ollama through its API (`OLLAMA_BASE_URL`).

- **Labels**:
  - `"traefik.http.routers.open-webui.rule=HostRegexp(^${HOST_NAME_PATTERN}\.(${LOCAL_DOMAIN_PATTERN}|${PUBLIC_DOMAIN_PATTERN})$)"` specifies the hostname pattern for accessing the service.

## Troubleshooting
- **Cloudflare Tunnel not connecting**: Ensure your `TUNNEL_TOKEN` is correct and that your Cloudflare account is properly set up.
- **SSL Issues**: If Let's Encrypt SSL certificates are not being issued, verify your Cloudflare API credentials and ensure DNS settings for your domain are correctly configured.

## Useful Resources
- [Traefik Documentation](https://doc.traefik.io/traefik/)
- [Cloudflare Tunnel Documentation](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/)
- [Ollama Engine Docker Setup](https://ollama.com/blog/ollama-is-now-available-as-an-official-docker-image)
