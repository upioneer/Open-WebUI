# Connecting Open WebUI (Docker) to a Native Ollama Installation

This guide walks you through setting up Open WebUI with Docker to connect to a native Ollama instance on Ubuntu or WSL.

---

## Prerequisites

This guide assumes you have the following:

-   An Ubuntu system (LTS versions like 22.04 or 24.04 recommended).
-   `sudo` or root privileges.
-   Ollama already installed and running natively on the host system.
-   (Optional) An NVIDIA GPU with the appropriate drivers installed.

---

## Install Docker

Update your system's package lists.
```bash
sudo apt update && sudo apt upgrade -y
```

Install prerequisite packages for Docker.
```bash
sudo apt-get install -y ca-certificates curl gnupg
```

Add Docker's official GPG key to ensure package authenticity.
```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL [https://download.docker.com/linux/ubuntu/gpg](https://download.docker.com/linux/ubuntu/gpg) | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

Set up the official Docker repository in your package sources.
```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] [https://download.docker.com/linux/ubuntu](https://download.docker.com/linux/ubuntu) \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Install Docker Engine and Docker Compose.
```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

---

## (Optional) Install NVIDIA Container Toolkit

**If you do not have an NVIDIA GPU, skip this section.** This toolkit allows Docker containers to access your GPU.

Add the NVIDIA package repository.
```bash
curl -fsSL [https://nvidia.github.io/libnvidia-container/gpgkey](https://nvidia.github.io/libnvidia-container/gpgkey) | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L [https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list](https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list) | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
```

Install the toolkit.
```bash
sudo apt update
sudo apt install -y nvidia-container-toolkit
```

Configure Docker to use the NVIDIA runtime and restart the service.
```bash
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

---

## Configure and Launch Open WebUI

Create a project directory to hold your configuration.
```bash
mkdir open-webui && cd open-webui
```

Create the `docker-compose.yml` file.
```bash
nano docker-compose.yml
```

Paste the following content into the file. This tells Docker how to run Open WebUI and connect it to your native Ollama instance.
```yaml
services:
  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: open-webui
    ports:
      # Exposes the Web UI on port 3000 of your local machine
      - "127.0.0.1:3000:8080"
    environment:
      # Tells WebUI to connect to Ollama on the host machine
      - 'OLLAMA_BASE_URL=[http://host.docker.internal:11434](http://host.docker.internal:11434)'
    volumes:
      # Persists Web UI data (users, chats, settings)
      - open-webui:/app/backend/data
    # Required for the container to resolve 'host.docker.internal'
    extra_hosts:
      - "host.docker.internal:host-gateway"
    restart: unless-stopped

volumes:
  open-webui:
```

Launch the Open WebUI container in the background.
```bash
sudo docker compose up -d
```
You can now access the interface by opening a web browser to **`http://localhost:3000`**.

---

## Troubleshooting: Models Not Detected

This is the most common issue. The Web UI loads, but the model list is empty because the native Ollama service isn't configured to accept external connections.

### Diagnosis

Check which address Ollama is using to listen for connections.
```bash
ss -ltn | grep 11434
```
-   **Incorrect Output:** `127.0.0.1:11434`. This confirms Ollama is only listening locally.
-   **Correct Output:** `0.0.0.0:11434` or `*:11434`. This means Ollama is correctly listening on all network interfaces.

### The Solution

Configure the Ollama `systemd` service to listen on all interfaces permanently.

1.  **Edit the Ollama service configuration.** This command safely creates an override file.
    ```bash
    sudo systemctl edit ollama.service
    ```

2.  **Add the environment variable.** An empty editor will open. Paste the following lines to tell the service to listen on all hosts.
    ```ini
    [Service]
    Environment="OLLAMA_HOST=0.0.0.0"
    ```
    Save and exit the editor (`CTRL+X`, `Y`, `Enter`).

3.  **Apply the changes and restart Ollama.**
    ```bash
    sudo systemctl daemon-reload
    sudo systemctl restart ollama.service
    ```

4.  **Verify the fix.** Run the diagnosis command again. The output should now show `*:11434` or `0.0.0.0:11434`.
    ```bash
    ss -ltn | grep 11434
    ```

5.  **Refresh the Web UI.** Go to `http://localhost:3000` in your browser and perform a hard refresh (`Ctrl+Shift+R`). Your models will now appear.

---

## Basic Management Commands

Run these commands from within your `open-webui` project directory.

Stop the Web UI:
```bash
sudo docker compose down
```

Start the Web UI:
```bash
sudo docker compose up -d
```

Update the Web UI to the latest version:
```bash
sudo docker compose pull
sudo docker compose up -d
```

View the logs for troubleshooting:
```bash
sudo docker compose logs -f
