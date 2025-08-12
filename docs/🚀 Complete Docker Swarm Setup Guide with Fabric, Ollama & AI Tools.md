# 🚀 Complete Docker Swarm Setup Guide with Fabric, Ollama & AI Tools

## 📋 Table of Contents

1. [🎯 Overview](#-overview)
2. [⚙️ Prerequisites](#️-prerequisites)
3. [🔧 Host System Setup](#-host-system-setup)
4. [🐳 Docker Environment Setup](#-docker-environment-setup)
5. [🌐 Tailscale Zero Trust VPN Setup](#-tailscale-zero-trust-vpn-setup)
6. [🔐 SSH & Git Server Configuration](#-ssh--git-server-configuration)
7. [🤖 Fabric Installation & Configuration](#-fabric-installation--configuration)
8. [🧠 Ollama AI Integration](#-ollama-ai-integration)
9. [📱 Msty.app Integration](#-mstyapp-integration)
10. [🎨 VSCode & Development Environment](#-vscode--development-environment)
11. [🔄 Docker Container Configurations](#-docker-container-configurations)
12. [🌊 Docker Swarm Orchestration](#-docker-swarm-orchestration)
13. [💾 Volume Management & Storage](#-volume-management--storage)
14. [📊 Monitoring & Maintenance](#-monitoring--maintenance)
15. [🚨 Troubleshooting](#-troubleshooting)

---

## 🎯 Overview

This guide provides a complete setup for a 5-node Docker Swarm cluster with AI development tools, optimized for:
- **Windows 11 Pro** with WSL2 Ubuntu 24.04 LTS
- **NVIDIA RTX 4090/5090** GPU acceleration
- **4TB D: SSD** for local operations
- **8TB E: SATA** for data volumes
- **Synology DS920+** for network storage
- **Tailscale** for zero-trust networking

### 🏗️ Architecture Overview

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Manager Node  │    │   Worker Node   │    │   Worker Node   │
│   (Windows 11)  │    │   (Ubuntu 24)   │    │   (Ubuntu 24)   │
│   RTX 4090/5090 │    │                 │    │                 │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         └───────────────────────┼───────────────────────┘
                                 │
                    ┌─────────────────┐
                    │  Tailscale VPN  │
                    │  Zero Trust Net │
                    └─────────────────┘
                                 │
                    ┌─────────────────┐
                    │ Synology DS920+ │
                    │ Network Storage │
                    └─────────────────┘
```

[⬆️ Back to TOC](#-table-of-contents)

---

## ⚙️ Prerequisites

### 📋 Hardware Requirements

| Component | Specification |
|-----------|---------------|
| **CPU** | Intel i7/i9 or AMD Ryzen 7/9 |
| **RAM** | 32GB+ DDR4/DDR5 |
| **GPU** | NVIDIA RTX 4090 or RTX 5090 |
| **Storage** | 4TB NVMe SSD (D:) + 8TB SATA (E:) |
| **Network** | Gigabit Ethernet minimum |

### 💿 Software Versions

| Software | Version | Purpose |
|----------|---------|---------|
| **Windows 11 Pro** | Latest (24H2) | Host OS |
| **WSL2** | Latest | Linux subsystem |
| **Ubuntu** | 24.04 LTS | WSL2 distribution |
| **Docker Desktop** | 4.26+ | Container runtime |
| **NVIDIA Container Toolkit** | 1.14+ | GPU support |
| **Python** | 3.13+ | Development |
| **Go** | 1.21+ | Fabric compilation |
| **Git** | 2.43+ | Version control |
| **Tailscale** | 1.56+ | Zero-trust VPN |

[⬆️ Back to TOC](#-table-of-contents)

---

## 🔧 Host System Setup

### 🪟 Windows 11 Pro Configuration

#### 1. Enable WSL2 and Hyper-V

```powershell
# Run as Administrator
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All

# Restart required
Restart-Computer
```

#### 2. Install WSL2 Ubuntu 24.04

```powershell
# Set WSL2 as default
wsl --set-default-version 2

# Install Ubuntu 24.04
wsl --install -d Ubuntu-24.04

# Set as default distribution
wsl --set-default Ubuntu-24.04
```

#### 3. Configure Drive Mapping

```bash
# In WSL2 - Create mount points
sudo mkdir -p /mnt/d /mnt/e /mnt/synology

# Add to /etc/fstab for persistent mounting
echo "D: /mnt/d drvfs defaults 0 0" | sudo tee -a /etc/fstab
echo "E: /mnt/e drvfs defaults 0 0" | sudo tee -a /etc/fstab
```

### 🐧 Ubuntu 24.04 LTS Setup

#### 1. System Update and Essential Packages

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install essential packages
sudo apt install -y \
    curl \
    wget \
    git \
    build-essential \
    software-properties-common \
    apt-transport-https \
    ca-certificates \
    gnupg \
    lsb-release \
    unzip \
    jq \
    htop \
    tree \
    vim \
    openssh-server \
    fail2ban
```

#### 2. NVIDIA Driver and CUDA Setup

```bash
# Install NVIDIA drivers (latest compatible)
sudo apt install -y nvidia-driver-545

# Add NVIDIA package repositories
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2404/x86_64/cuda-keyring_1.1-1_all.deb
sudo dpkg -i cuda-keyring_1.1-1_all.deb
sudo apt update

# Install CUDA Toolkit 13.0
sudo apt install -y cuda-toolkit-13-0

# Add to PATH
echo 'export PATH=/usr/local/cuda-13.0/bin:$PATH' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH=/usr/local/cuda-13.0/lib64:$LD_LIBRARY_PATH' >> ~/.bashrc
source ~/.bashrc
```

#### 3. Python 3.13+ Installation

```bash
# Add deadsnakes PPA for latest Python
sudo add-apt-repository ppa:deadsnakes/ppa -y
sudo apt update

# Install Python 3.13
sudo apt install -y python3.13 python3.13-venv python3.13-dev python3-pip

# Set Python 3.13 as default
sudo update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.13 1

# Install UV (Astral's package manager)
curl -LsSf https://astral.sh/uv/install.sh | sh
source ~/.bashrc
```

#### 4. Go Installation (for Fabric)

```bash
# Download and install Go 1.21+
cd /tmp
wget https://go.dev/dl/go1.21.5.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.21.5.linux-amd64.tar.gz

# Add Go to PATH
echo 'export GOROOT=/usr/local/go' >> ~/.bashrc
echo 'export GOPATH=$HOME/go' >> ~/.bashrc
echo 'export PATH=$GOPATH/bin:$GOROOT/bin:$HOME/.local/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

[⬆️ Back to TOC](#-table-of-contents)

---

## 🐳 Docker Environment Setup

### 🪟 Docker Desktop Installation (Windows)

#### 1. Install Docker Desktop

```powershell
# Download and install Docker Desktop with WSL2 backend
# From: https://desktop.docker.com/win/main/amd64/Docker%20Desktop%20Installer.exe

# Configure Docker Desktop settings:
# - Enable WSL2 integration
# - Enable Ubuntu-24.04 integration
# - Allocate adequate resources (16GB+ RAM, 8+ CPU cores)
```

### 🐧 Docker Engine Setup (Ubuntu Nodes)

#### 1. Install Docker Engine

```bash
# Add Docker's official GPG key
sudo apt update
sudo apt install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add repository
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker Engine
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Add user to docker group
sudo usermod -aG docker $USER
newgrp docker
```

#### 2. NVIDIA Container Toolkit

```bash
# Configure repository
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt update

# Install toolkit
sudo apt install -y nvidia-container-toolkit

# Configure Docker daemon
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker

# Test GPU access
docker run --rm --runtime=nvidia --gpus all nvidia/cuda:13.0-base-ubuntu24.04 nvidia-smi
```

#### 3. Docker Daemon Configuration

```bash
# Create daemon configuration
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json > /dev/null <<EOF
{
  "default-runtime": "nvidia",
  "runtimes": {
    "nvidia": {
      "path": "nvidia-container-runtime",
      "runtimeArgs": []
    }
  },
  "default-shm-size": "1G",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
EOF

# Restart Docker
sudo systemctl restart docker
sudo systemctl enable docker
```

[⬆️ Back to TOC](#-table-of-contents)

---

## 🌐 Tailscale Zero Trust VPN Setup

### 📡 Installation on All Nodes

#### 1. Windows 11 Installation

```powershell
# Download and install Tailscale for Windows
# From: https://tailscale.com/download/windows

# Or via winget
winget install tailscale.tailscale
```

#### 2. Ubuntu Installation

```bash
# Add Tailscale repository
curl -fsSL https://pkgs.tailscale.com/stable/ubuntu/noble.noarmor.gpg | sudo tee /usr/share/keyrings/tailscale-archive-keyring.gpg >/dev/null
curl -fsSL https://pkgs.tailscale.com/stable/ubuntu/noble.tailscale-keyring.list | sudo tee /etc/apt/sources.list.d/tailscale.list

sudo apt update
sudo apt install tailscale

# Start and enable service
sudo systemctl enable --now tailscaled

# Connect to Tailscale network
sudo tailscale up

# Enable subnet routing (if needed)
sudo tailscale up --advertise-routes=192.168.1.0/24 --accept-dns=false
```

### 🔧 Network Configuration

#### 1. Configure ACL (Access Control List)

```json
// In Tailscale Admin Console -> Access Controls
{
  "acls": [
    {
      "action": "accept",
      "src": ["autogroup:members"],
      "dst": ["*:22", "*:2376", "*:2377", "*:7946", "*:4789"]
    }
  ],
  "ssh": [
    {
      "action": "accept",
      "src": ["autogroup:members"],
      "dst": ["autogroup:self"],
      "users": ["autogroup:nonroot", "root"]
    }
  ]
}
```

#### 2. Enable Magic DNS and HTTPS

```bash
# Configure DNS settings in Tailscale admin
# Enable MagicDNS
# Enable HTTPS certificates

# Test connectivity between nodes
tailscale ping node-name
```

[⬆️ Back to TOC](#-table-of-contents)

---

## 🔐 SSH & Git Server Configuration

### 🔑 SSH Key Setup

#### 1. Generate SSH Keys

```bash
# Generate Ed25519 key pair
ssh-keygen -t ed25519 -C "docker-swarm@yourdomain.com" -f ~/.ssh/docker_swarm_ed25519

# Start SSH agent
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/docker_swarm_ed25519
```

#### 2. Distribute Keys to All Nodes

```bash
# Copy public key to all nodes
for node in node1 node2 node3 node4 node5; do
    ssh-copy-id -i ~/.ssh/docker_swarm_ed25519.pub user@$node
done

# Test passwordless SSH
ssh -i ~/.ssh/docker_swarm_ed25519 user@node1
```

### 🌐 Git Server Setup

#### 1. Bare Git Repository

```bash
# Create shared git repository on manager node
sudo mkdir -p /mnt/d/git-repos
sudo chown -R $USER:docker /mnt/d/git-repos

# Initialize bare repository for Fabric patterns
cd /mnt/d/git-repos
git init --bare fabric-patterns.git

# Clone for local development
cd /mnt/d/development
git clone /mnt/d/git-repos/fabric-patterns.git
cd fabric-patterns

# Initial commit
echo "# Fabric Custom Patterns" > README.md
git add README.md
git commit -m "Initial commit"
git push origin main
```

#### 2. Git Configuration

```bash
# Configure Git globally
git config --global user.name "Docker Swarm User"
git config --global user.email "docker-swarm@yourdomain.com"
git config --global init.defaultBranch main
git config --global push.default simple

# Configure Git for Fabric patterns
git config --global fabric.patterns.path "/mnt/d/fabric-patterns"
```

[⬆️ Back to TOC](#-table-of-contents)

---

## 🤖 Fabric Installation & Configuration

### 📥 Fabric Installation

#### 1. Install from Latest Release

```bash
# Create Fabric directory
mkdir -p /mnt/d/fabric/{bin,patterns,sessions}

# Download latest Fabric binary
curl -L https://github.com/danielmiessler/fabric/releases/latest/download/fabric-linux-amd64 > /mnt/d/fabric/bin/fabric
chmod +x /mnt/d/fabric/bin/fabric

# Create symbolic link
sudo ln -sf /mnt/d/fabric/bin/fabric /usr/local/bin/fabric

# Verify installation
fabric --version
```

#### 2. Alternative: Install from Source

```bash
# Install directly from repository
go install github.com/danielmiessler/fabric@latest

# Verify Go installation
which fabric
fabric --version
```

### ⚙️ Fabric Configuration

#### 1. Environment Setup

```bash
# Create Fabric configuration
mkdir -p ~/.config/fabric

# Fabric configuration file
tee ~/.config/fabric/config.yaml > /dev/null <<EOF
# Fabric Configuration
patterns_path: "/mnt/d/fabric/patterns"
sessions_path: "/mnt/d/fabric/sessions"
models_path: "/mnt/d/fabric/models"

# Default model settings
default_model: "llama3.1:70b"
temperature: 0.7
max_tokens: 4000

# API configurations
openai:
  api_key: "${OPENAI_API_KEY}"
  base_url: "https://api.openai.com/v1"

ollama:
  base_url: "http://localhost:11434"
  
anthropic:
  api_key: "${ANTHROPIC_API_KEY}"
EOF
```

#### 2. Custom Patterns Setup

```bash
# Create custom patterns directory structure
mkdir -p /mnt/d/fabric/patterns/{
    code-analysis,
    docker-optimization,
    ai-training,
    system-monitoring,
    security-audit
}

# Example custom pattern for Docker optimization
tee /mnt/d/fabric/patterns/docker-optimization/system.md > /dev/null <<EOF
# IDENTITY and PURPOSE

You are an expert Docker and container optimization specialist. You analyze Docker configurations, Dockerfiles, and container deployments to provide specific optimization recommendations.

# STEPS

- Analyze the provided Docker configuration or Dockerfile
- Identify performance bottlenecks and security issues
- Recommend specific optimizations for size, speed, and security
- Provide updated configuration with explanations

# OUTPUT INSTRUCTIONS

- Provide specific, actionable recommendations
- Include optimized Dockerfile or docker-compose.yml
- Explain the reasoning behind each optimization
- Consider security implications of all changes

# INPUT

INPUT:
EOF
```

#### 3. Auto-Update Script

```bash
# Create daily update script
tee /mnt/d/fabric/bin/update-fabric.sh > /dev/null <<EOF
#!/bin/bash
# Auto-update Fabric to latest version

set -euo pipefail

FABRIC_BIN="/mnt/d/fabric/bin/fabric"
BACKUP_DIR="/mnt/d/fabric/backups"
LOG_FILE="/mnt/d/fabric/logs/update.log"

# Create directories
mkdir -p "\$BACKUP_DIR" "\$(dirname "\$LOG_FILE")"

# Backup current version
if [ -f "\$FABRIC_BIN" ]; then
    cp "\$FABRIC_BIN" "\$BACKUP_DIR/fabric-\$(date +%Y%m%d-%H%M%S)"
fi

# Download latest version
echo "\$(date): Updating Fabric to latest version..." >> "\$LOG_FILE"
curl -L https://github.com/danielmiessler/fabric/releases/latest/download/fabric-linux-amd64 > "\$FABRIC_BIN.new"

# Verify download
if [ -f "\$FABRIC_BIN.new" ] && [ -s "\$FABRIC_BIN.new" ]; then
    chmod +x "\$FABRIC_BIN.new"
    mv "\$FABRIC_BIN.new" "\$FABRIC_BIN"
    echo "\$(date): Fabric updated successfully" >> "\$LOG_FILE"
else
    echo "\$(date): Failed to update Fabric" >> "\$LOG_FILE"
    exit 1
fi

# Test new version
"\$FABRIC_BIN" --version >> "\$LOG_FILE" 2>&1
EOF

chmod +x /mnt/d/fabric/bin/update-fabric.sh

# Add to cron for daily updates
(crontab -l 2>/dev/null; echo "0 2 * * * /mnt/d/fabric/bin/update-fabric.sh") | crontab -
```

[⬆️ Back to TOC](#-table-of-contents)

---

## 🧠 Ollama AI Integration

### 📦 Ollama Installation & Setup

#### 1. Pull Ollama Docker Image

```bash
# Pull latest Ollama image
docker pull ollama/ollama:latest

# Pull CUDA-enabled base image
docker pull nvidia/cuda:13.0-cudnn8-devel-ubuntu24.04
```

#### 2. Create Ollama Data Directories

```bash
# Create persistent data directories
mkdir -p /mnt/d/ollama/{models,config,logs}
mkdir -p /mnt/e/ollama-data/{models,cache}

# Set permissions
sudo chown -R 1000:1000 /mnt/d/ollama /mnt/e/ollama-data
```

#### 3. Ollama Configuration

```bash
# Create Ollama configuration file
tee /mnt/d/ollama/config/config.json > /dev/null <<EOF
{
  "model_path": "/mnt/e/ollama-data/models",
  "host": "0.0.0.0:11434",
  "origins": ["*"],
  "gpu_memory_fraction": 0.9,
  "concurrent_requests": 4,
  "cache_dir": "/mnt/e/ollama-data/cache",
  "log_level": "info"
}
EOF
```

### 🚀 Ollama Auto-Update Script

```bash
# Create Ollama update script
tee /mnt/d/ollama/bin/update-ollama.sh > /dev/null <<EOF
#!/bin/bash
# Auto-update Ollama and models

set -euo pipefail

LOG_FILE="/mnt/d/ollama/logs/update.log"
mkdir -p "\$(dirname "\$LOG_FILE")"

echo "\$(date): Starting Ollama update process..." >> "\$LOG_FILE"

# Update Ollama container
echo "\$(date): Pulling latest Ollama image..." >> "\$LOG_FILE"
docker pull ollama/ollama:latest >> "\$LOG_FILE" 2>&1

# Check for model updates
MODELS=("llama3.1:70b" "llama3.1:8b" "codellama:13b" "phi3:medium")

for model in "\${MODELS[@]}"; do
    echo "\$(date): Checking updates for \$model..." >> "\$LOG_FILE"
    docker exec ollama-server ollama pull "\$model" >> "\$LOG_FILE" 2>&1 || true
done

echo "\$(date): Ollama update process completed." >> "\$LOG_FILE"
EOF

chmod +x /mnt/d/ollama/bin/update-ollama.sh

# Add to cron (daily at 3 AM)
(crontab -l 2>/dev/null; echo "0 3 * * * /mnt/d/ollama/bin/update-ollama.sh") | crontab -
```

### 📋 Model Management

```bash
# Create model management script
tee /mnt/d/ollama/bin/manage-models.sh > /dev/null <<EOF
#!/bin/bash
# Manage Ollama models

MODELS_CONFIG="/mnt/d/ollama/config/models.json"

# Default models configuration
tee "\$MODELS_CONFIG" > /dev/null <<JSON
{
  "required_models": [
    {
      "name": "llama3.1:70b",
      "description": "Large general-purpose model",
      "size": "~40GB"
    },
    {
      "name": "llama3.1:8b", 
      "description": "Fast general-purpose model",
      "size": "~4.7GB"
    },
    {
      "name": "codellama:13b",
      "description": "Code generation and analysis",
      "size": "~7.3GB"
    },
    {
      "name": "phi3:medium",
      "description": "Efficient small model",
      "size": "~7.9GB"
    }
  ],
  "custom_models": [
    {
      "name": "fabric-assistant",
      "base": "llama3.1:8b",
      "description": "Custom model for Fabric patterns"
    }
  ]
}
JSON

# Function to download required models
download_models() {
    echo "Downloading required models..."
    jq -r '.required_models[].name' "\$MODELS_CONFIG" | while read model; do
        echo "Downloading \$model..."
        docker exec ollama-server ollama pull "\$model"
    done
}

# Function to list available models
list_models() {
    echo "Available models:"
    docker exec ollama-server ollama list
}

# Function to remove unused models
cleanup_models() {
    echo "Cleaning up unused models..."
    # Add cleanup logic here
}

case "\${1:-help}" in
    download) download_models ;;
    list) list_models ;;
    cleanup) cleanup_models ;;
    *) 
        echo "Usage: \$0 {download|list|cleanup}"
        echo "  download - Download all required models"
        echo "  list     - List available models"
        echo "  cleanup  - Remove unused models"
        ;;
esac
EOF

chmod +x /mnt/d/ollama/bin/manage-models.sh
```

[⬆️ Back to TOC](#-table-of-contents)

---

## 📱 Msty.app Integration

### 🔧 Msty Configuration

#### 1. Msty Desktop Application Setup

```bash
# Create Msty configuration directory
mkdir -p /mnt/d/msty/{config,themes,plugins}

# Msty configuration file
tee /mnt/d/msty/config/msty-config.json > /dev/null <<EOF
{
  "app": {
    "theme": "dark",
    "auto_start": true,
    "minimize_to_tray": true
  },
  "connections": [
    {
      "name": "Local Ollama",
      "type": "ollama",
      "url": "http://localhost:11434",
      "default": true
    },
    {
      "name": "Docker Swarm Ollama",
      "type": "ollama", 
      "url": "http://ollama-swarm:11434",
      "load_balance": true
    }
  ],
  "models": {
    "preferred": ["llama3.1:70b", "llama3.1:8b"],
    "auto_load": true,
    "gpu_acceleration": true
  },
  "ui": {
    "conversation_history": "/mnt/d/msty/conversations",
    "export_formats": ["markdown", "json", "txt"],
    "syntax_highlighting": true
  },
  "integrations": {
    "fabric": {
      "enabled": true,
      "patterns_path": "/mnt/d/fabric/patterns",
      "auto_apply_patterns": false
    },
    "vscode": {
      "enabled": true,
      "workspace_integration": true
    }
  }
}
EOF
```

#### 2. Docker Integration Script

```bash
# Create Msty Docker integration script
tee /mnt/d/msty/bin/msty-docker.sh > /dev/null <<EOF
#!/bin/bash
# Msty Docker integration utilities

set -euo pipefail

MSTY_CONFIG="/mnt/d/msty/config/msty-config.json"
DOCKER_SOCKET="/var/run/docker.sock"

# Function to detect available Ollama instances
detect_ollama_instances() {
    echo "Detecting Ollama instances..."
    
    # Local instance
    if curl -s http://localhost:11434/api/tags > /dev/null 2>&1; then
        echo "✓ Local Ollama instance detected on port 11434"
    fi
    
    # Docker Swarm instances
    docker service ls --filter name=ollama --format "table {{.Name}}\t{{.Replicas}}\t{{.Ports}}" 2>/dev/null || true
}

# Function to update Msty configuration with available instances
update_msty_config() {
    echo "Updating Msty configuration..."
    
    # Backup current config
    cp "\$MSTY_CONFIG" "\$MSTY_CONFIG.backup.\$(date +%Y%m%d-%H%M%S)"
    
    # Update configuration with discovered instances
    # (Implementation would depend on jq manipulation)
}

# Function to test Msty connections
test_connections() {
    echo "Testing Msty connections..."
    
    # Test each configured endpoint
    jq -r '.connections[].url' "\$MSTY_CONFIG" | while read url; do
        if curl -s "\$url/api/tags" > /dev/null 2>&1; then
            echo "✓ \$url - Connection successful"
        else
            echo "✗ \$url - Connection failed"
        fi
    done
}

case "\${1:-help}" in
    detect) detect_ollama_instances ;;
    update) update_msty_config ;;
    test) test_connections ;;
    *)
        echo "Usage: \$0 {detect|update|test}"
        echo "  detect - Detect available Ollama instances"
        echo "  update - Update Msty configuration"
        echo "  test   - Test configured connections"
        ;;
esac
EOF

chmod +x /mnt/d/msty/bin/msty-docker.sh
```

[⬆️ Back to TOC](#-table-of-contents)

---

## 🎨 VSCode & Development Environment

### 💻 VSCode Configuration

#### 1. VSCode Extensions

```json
// /mnt/d/vscode/extensions/extensions.json
{
  "recommendations": [
    "ms-python.python",
    "ms-python.vscode-pylance", 
    "ms-toolsai.jupyter",
    "ms-vscode-remote.remote-containers",
    "ms-vscode-remote.remote-wsl",
    "ms-vscode.docker",
    "github.copilot",
    "github.copilot-chat",
    "redhat.vscode-yaml",
    "ms-kubernetes-tools.vscode-kubernetes-tools",
    "ms-vscode.go",
    "rust-lang.rust-analyzer",
    "bradlc.vscode-tailwindcss",
    "esbenp.prettier-vscode",
    "ms-vsliveshare.vsliveshare"
  ]
}
```

#### 2. VSCode Settings

```json
// /mnt/d/vscode/settings/settings.json
{
  "python.defaultInterpreterPath": "/usr/bin/python3.13",
  "python.terminal.activateEnvironment": true,
  "python.analysis.typeCheckingMode": "strict",
  "python.analysis.autoImportCompletions": true,
  
  "docker.showStartPage": false,
  "docker.dockerPath": "docker",
  "docker.enableDockerComposeLanguageService": true,
  
  "remote.WSL.fileWatcher.polling": true,
  "remote.WSL.useShellEnvironment": true,
  
  "terminal.integrated.defaultProfile.linux": "bash",
  "terminal.integrated.profiles.linux": {
    "bash": {
      "path": "/bin/bash",
      "args": ["-l"]
    }
  },
  
  "files.watcherExclude": {
    "**/node_modules/**": true,
    "**/.git/objects/**": true,
    "**/ollama-data/**": true,
    "**/docker-volumes/**": true
  },
  
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "source.organizeImports": true
  },
  
  "go.toolsManagement.autoUpdate": true,
  "go.useLanguageServer": true,
  
  "jupyter.askForKernelRestart": false,
  "jupyter.interactiveWindow.creationMode": "perFile"
}
```

#### 3. Development Workspace

```json
// /mnt/d/vscode/workspaces/docker-swarm-ai.code-workspace
{
  "folders": [
    {
      "name": "Fabric Patterns",
      "path": "/mnt/d/fabric/patterns"
    },
    {
      "name": "Docker Configurations",
      "path": "/mnt/d/docker-configs"
    },
    {
      "name": "Python Development",
      "path": "/mnt/d/python-projects"
    },
    {
      "name": "Ollama Models",
      "path": "/mnt/e/ollama-data"
    },
    {
      "name": "Shared Storage",
      "path": "/mnt/synology"
    }
  ],
  "settings": {
    "python.analysis.extraPaths": [
      "/mnt/d/python-projects/shared-libs"
    ],
    "docker.host": "unix:///var/run/docker.sock",
    "remote.containers.defaultExtensions": [
      "ms-python.python",
      "ms-toolsai.jupyter"
    ]
  },
  "extensions": {
    "recommendations": [
      "ms-python.python",
      "ms-vscode.docker",
      "ms-vscode-remote.remote-containers"
    ]
  }
}
```

### 🐍 Python Environment Setup

#### 1. UV (Astral) Configuration

```bash
# Create UV configuration
mkdir -p /mnt/d/python-projects/{shared-libs,ai-tools,fabric-extensions}

# UV project configuration
tee /mnt/d/python-projects/pyproject.toml > /dev/null <<EOF
[project]
name = "docker-swarm-ai-tools"
version = "0.1.0"
description = "AI development tools for Docker Swarm environment"
authors = [{name = "Docker Swarm User", email = "user@example.com"}]
dependencies = [
    "jupyter>=1.0.0",
    "marimo>=0.3.0",
    "numpy>=1.24.0",
    "pandas>=2.0.0",
    "scikit-learn>=1.3.0",
    "torch>=2.1.0",
    "transformers>=4.35.0",
    "ollama>=0.1.8",
    "docker>=6.1.0",
    "fabric-ai>=1.0.0",
    "fastapi>=0.104.0",
    "uvicorn>=0.24.0",
    "pydantic>=2.4.0",
    "httpx>=0.25.0",
    "python-multipart>=0.0.6",
    "python-dotenv>=1.0.0"
]
requires-python = ">=3.13"

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.uv]
dev-dependencies = [
    "pytest>=7.4.0",
    "black>=23.9.0",
    "isort>=5.12.0",
    "mypy>=1.6.0",
    "pre-commit>=3.4.0"
]

[tool.black]
line-length = 88
target-version = ['py313']

[tool.isort]
profile = "black"
line_length = 88
EOF

# Initialize UV environment
cd /mnt/d/python-projects
uv init
uv add --dev pytest black isort mypy
uv sync
```

#### 2. Marimo Notebooks Setup

```bash
# Create Marimo configuration
mkdir -p /mnt/d/python-projects/marimo-notebooks/{ai-analysis,docker-monitoring,fabric-workflows}

# Sample Marimo notebook for AI analysis
tee /mnt/d/python-projects/marimo-notebooks/ai-analysis/ollama_analysis.py > /dev/null <<'EOF'
import marimo

__generated_with = "0.3.8"
app = marimo.App()

@app.cell
def __():
    import marimo as mo
    import pandas as pd
    import numpy as np
    import requests
    import json
    from datetime import datetime
    return datetime, json, mo, np, pd, requests

@app.cell
def __(mo):
    mo.md("""
    # Ollama Model Analysis Dashboard
    
    This notebook provides analysis and monitoring of Ollama models in the Docker Swarm environment.
    """)
    return

@app.cell
def __(requests, json):
    def get_ollama_models(base_url="http://localhost:11434"):
        """Get list of available models from Ollama"""
        try:
            response = requests.get(f"{base_url}/api/tags")
            return response.json()
        except Exception as e:
            return {"error": str(e)}
    
    def query_model(model_name, prompt, base_url="http://localhost:11434"):
        """Query a specific model with a prompt"""
        data = {
            "model": model_name,
            "prompt": prompt,
            "stream": False
        }
        try:
            response = requests.post(f"{base_url}/api/generate", json=data)
            return response.json()
        except Exception as e:
            return {"error": str(e)}
    
    return get_ollama_models, query_model

@app.cell
def __(get_ollama_models, mo, pd):
    # Get available models
    models_data = get_ollama_models()
    
    if "error" not in models_data and "models" in models_data:
        models_df = pd.DataFrame(models_data["models"])
        mo.ui.table(models_df)
    else:
        mo.md("**Error connecting to Ollama instance**")
    return models_data, models_df

if __name__ == "__main__":
    app.run()
EOF

# Create Marimo startup script
tee /mnt/d/python-projects/bin/start-marimo.sh > /dev/null <<EOF
#!/bin/bash
# Start Marimo server with Docker Swarm integration

cd /mnt/d/python-projects
source .venv/bin/activate

# Start Marimo server
marimo edit \
    --host 0.0.0.0 \
    --port 7860 \
    --no-token \
    marimo-notebooks/
EOF

chmod +x /mnt/d/python-projects/bin/start-marimo.sh
```

[⬆️ Back to TOC](#-table-of-contents)

---

## 🔄 Docker Container Configurations

### 🏗️ Multi-Stage Dockerfile for AI Development

#### 1. Base AI Development Container

```dockerfile
# /mnt/d/docker-configs/Dockerfile.ai-dev
# Multi-stage build for AI development with Fabric, Ollama, and Python 3.13

# Stage 1: Base Ubuntu with CUDA support
FROM nvidia/cuda:13.0-cudnn8-devel-ubuntu24.04 AS base

# Set environment variables
ENV DEBIAN_FRONTEND=noninteractive
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1
ENV PATH="/usr/local/go/bin:/opt/uv/bin:$PATH"

# Install system dependencies
RUN apt-get update && apt-get install -y \
    curl \
    wget \
    git \
    build-essential \
    software-properties-common \
    ca-certificates \
    gnupg \
    lsb-release \
    unzip \
    jq \
    htop \
    vim \
    openssh-client \
    && rm -rf /var/lib/apt/lists/*

# Install Python 3.13
RUN add-apt-repository ppa:deadsnakes/ppa -y && \
    apt-get update && \
    apt-get install -y python3.13 python3.13-venv python3.13-dev python3-pip && \
    update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.13 1 && \
    rm -rf /var/lib/apt/lists/*

# Install Go for Fabric
RUN wget -O - https://go.dev/dl/go1.21.5.linux-amd64.tar.gz | tar -C /usr/local -xzf -

# Install UV (Astral package manager)
RUN curl -LsSf https://astral.sh/uv/install.sh | sh

# Stage 2: Development tools
FROM base AS dev-tools

# Install Node.js (for some development tools)
RUN curl -fsSL https://deb.nodesource.com/setup_lts.x | bash - && \
    apt-get install -y nodejs && \
    rm -rf /var/lib/apt/lists/*

# Install Fabric from source
RUN go install github.com/danielmiessler/fabric@latest

# Create working directories
RUN mkdir -p /app/{fabric,ollama,python-projects,notebooks}
WORKDIR /app

# Stage 3: Python environment setup
FROM dev-tools AS python-env

# Copy Python project configuration
COPY pyproject.toml uv.lock* ./

# Create virtual environment and install dependencies
RUN uv venv .venv && \
    . .venv/bin/activate && \
    uv pip install -e .

# Stage 4: Final production image
FROM python-env AS production

# Create non-root user
RUN useradd -m -s /bin/bash aidev && \
    chown -R aidev:aidev /app

USER aidev

# Set up environment
ENV PATH="/app/.venv/bin:$PATH"
ENV FABRIC_PATTERNS_PATH="/app/fabric/patterns"
ENV OLLAMA_HOST="http://ollama:11434"

# Copy application files
COPY --chown=aidev:aidev fabric/ /app/fabric/
COPY --chown=aidev:aidev python-projects/ /app/python-projects/
COPY --chown=aidev:aidev scripts/ /app/scripts/

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

# Default command
CMD ["/app/scripts/start-services.sh"]
```

#### 2. Standalone Container (Auto-restart)

```dockerfile
# /mnt/d/docker-configs/Dockerfile.standalone
FROM nvidia/cuda:13.0-cudnn8-devel-ubuntu24.04

# Environment setup
ENV DEBIAN_FRONTEND=noninteractive
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

# Install system dependencies and Python 3.13
RUN apt-get update && apt-get install -y \
    curl wget git build-essential \
    software-properties-common ca-certificates \
    openssh-client supervisor \
    && add-apt-repository ppa:deadsnakes/ppa -y \
    && apt-get update && apt-get install -y \
    python3.13 python3.13-venv python3.13-dev \
    && rm -rf /var/lib/apt/lists/*

# Install Go and Fabric
RUN wget -O - https://go.dev/dl/go1.21.5.linux-amd64.tar.gz | tar -C /usr/local -xzf - \
    && export PATH="/usr/local/go/bin:$PATH" \
    && go install github.com/danielmiessler/fabric@latest

# Install UV and create Python environment
RUN curl -LsSf https://astral.sh/uv/install.sh | sh \
    && export PATH="/root/.local/bin:$PATH"

# Create application structure
RUN mkdir -p /app/{services,logs,data}
WORKDIR /app

# Copy configuration files
COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf
COPY entrypoint.sh /app/entrypoint.sh
RUN chmod +x /app/entrypoint.sh

# Supervisor configuration
RUN echo '[supervisord]\n\
nodaemon=true\n\
user=root\n\
logfile=/app/logs/supervisord.log\n\
pidfile=/var/run/supervisord.pid\n\
\n\
[program:fabric-server]\n\
command=/app/services/fabric-server.sh\n\
autostart=true\n\
autorestart=true\n\
stderr_logfile=/app/logs/fabric-server.err.log\n\
stdout_logfile=/app/logs/fabric-server.out.log\n\
\n\
[program:ollama-client]\n\
command=/app/services/ollama-client.sh\n\
autostart=true\n\
autorestart=true\n\
stderr_logfile=/app/logs/ollama-client.err.log\n\
stdout_logfile=/app/logs/ollama-client.out.log\n\
\n\
[program:marimo-server]\n\
command=/app/services/marimo-server.sh\n\
autostart=true\n\
autorestart=true\n\
stderr_logfile=/app/logs/marimo-server.err.log\n\
stdout_logfile=/app/logs/marimo-server.out.log' > /etc/supervisor/conf.d/supervisord.conf

EXPOSE 8000 7860 11434

CMD ["/usr/bin/supervisord", "-c", "/etc/supervisor/conf.d/supervisord.conf"]
```

#### 3. Docker Compose Configurations

```yaml
# /mnt/d/docker-configs/docker-compose.standalone.yml
version: '3.8'

services:
  ai-dev-standalone:
    build:
      context: .
      dockerfile: Dockerfile.standalone
      target: production
    container_name: ai-dev-standalone
    restart: unless-stopped
    ports:
      - "8000:8000"   # Main application
      - "7860:7860"   # Marimo notebooks
      - "11434:11434" # Ollama API proxy
    volumes:
      # D: drive mappings (local SSD)
      - /mnt/d/fabric:/app/fabric:rw
      - /mnt/d/python-projects:/app/python-projects:rw
      - /mnt/d/vscode/workspaces:/app/workspaces:ro
      - /mnt/d/git-repos:/app/git-repos:rw
      
      # E: drive mappings (data SATA)
      - /mnt/e/ollama-data:/app/ollama-data:rw
      - /mnt/e/docker-volumes:/app/volumes:rw
      
      # Synology mappings
      - /mnt/synology/shared:/app/shared:rw
      - /mnt/synology/models:/app/models:ro
      
      # System mounts
      - /var/run/docker.sock:/var/run/docker.sock:ro
    environment:
      - NVIDIA_VISIBLE_DEVICES=all
      - NVIDIA_DRIVER_CAPABILITIES=compute,utility
      - FABRIC_PATTERNS_PATH=/app/fabric/patterns
      - OLLAMA_HOST=http://ollama:11434
      - PYTHON_PATH=/app/python-projects
      - GIT_REPOS_PATH=/app/git-repos
      - SHARED_STORAGE_PATH=/app/shared
    runtime: nvidia
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
    networks:
      - ai-network

  ollama:
    image: ollama/ollama:latest
    container_name: ollama-server
    restart: unless-stopped
    ports:
      - "11435:11434"
    volumes:
      - /mnt/e/ollama-data:/root/.ollama:rw
      - /mnt/d/ollama/config:/config:ro
    environment:
      - NVIDIA_VISIBLE_DEVICES=all
      - NVIDIA_DRIVER_CAPABILITIES=compute,utility
      - OLLAMA_HOST=0.0.0.0:11434
    runtime: nvidia
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    networks:
      - ai-network

  msty-web:
    image: node:18-alpine
    container_name: msty-web
    restart: unless-stopped
    ports:
      - "3000:3000"
    volumes:
      - /mnt/d/msty:/app/msty:rw
    working_dir: /app/msty
    command: ["npm", "start"]
    depends_on:
      - ollama
    networks:
      - ai-network

networks:
  ai-network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16
```

#### 4. Docker Swarm Service Configuration

```yaml
# /mnt/d/docker-configs/docker-compose.swarm.yml
version: '3.8'

services:
  ai-dev-swarm:
    image: ai-dev:swarm
    build:
      context: .
      dockerfile: Dockerfile.ai-dev
      target: production
    ports:
      - target: 8000
        published: 8000
        protocol: tcp
        mode: ingress
      - target: 7860
        published: 7860
        protocol: tcp
        mode: ingress
    volumes:
      # Shared volumes across swarm
      - type: bind
        source: /mnt/d/fabric
        target: /app/fabric
      - type: bind
        source: /mnt/e/ollama-data
        target: /app/ollama-data
      - type: bind
        source: /mnt/synology/shared
        target: /app/shared
    environment:
      - NVIDIA_VISIBLE_DEVICES=all
      - FABRIC_PATTERNS_PATH=/app/fabric/patterns
      - SWARM_MODE=true
      - NODE_ROLE={{.Node.Role}}
      - SERVICE_NAME={{.Service.Name}}
    deploy:
      mode: replicated
      replicas: 3
      placement:
        constraints:
          - node.labels.gpu==true
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
        limits:
          memory: 16G
          cpus: '4.0'
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
        window: 120s
      update_config:
        parallelism: 1
        delay: 10s
        failure_action: rollback
        monitor: 60s
        order: stop-first
      rollback_config:
        parallelism: 1
        delay: 5s
        monitor: 60s
        order: stop-first
    networks:
      - swarm-network

  ollama-swarm:
    image: ollama/ollama:latest
    ports:
      - target: 11434
        published: 11434
        protocol: tcp
        mode: ingress
    volumes:
      - type: bind
        source: /mnt/e/ollama-data
        target: /root/.ollama
    environment:
      - NVIDIA_VISIBLE_DEVICES=all
      - OLLAMA_HOST=0.0.0.0:11434
    deploy:
      mode: global
      placement:
        constraints:
          - node.labels.gpu==true
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
      restart_policy:
        condition: on-failure
    networks:
      - swarm-network

  fabric-scheduler:
    image: ai-dev:swarm
    command: ["/app/scripts/fabric-scheduler.sh"]
    volumes:
      - type: bind
        source: /mnt/d/fabric
        target: /app/fabric
    environment:
      - SERVICE_TYPE=scheduler
      - SWARM_MODE=true
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints:
          - node.role==manager
      restart_policy:
        condition: on-failure
    networks:
      - swarm-network

networks:
  swarm-network:
    driver: overlay
    attachable: true
    ipam:
      config:
        - subnet: 10.0.0.0/24

volumes:
  fabric-patterns:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /mnt/d/fabric/patterns
  
  ollama-models:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /mnt/e/ollama-data/models
      
  synology-shared:
    driver: local
    driver_opts:
      type: nfs
      o: addr=synology-ds920.local,rw
      device: :/volume1/shared
```

[⬆️ Back to TOC](#-table-of-contents)

---

## 🌊 Docker Swarm Orchestration

### 🏗️ Swarm Initialization

#### 1. Initialize Docker Swarm

```bash
# On manager node (Windows WSL2 or primary Ubuntu node)
docker swarm init --advertise-addr $(tailscale ip --4)

# Save join tokens
WORKER_TOKEN=$(docker swarm join-token worker -q)
MANAGER_TOKEN=$(docker swarm join-token manager -q)

echo "Worker token: $WORKER_TOKEN" > /mnt/d/docker-configs/swarm-tokens.txt
echo "Manager token: $MANAGER_TOKEN" >> /mnt/d/docker-configs/swarm-tokens.txt
```

#### 2. Join Worker Nodes

```bash
# On each worker node
SWARM_MANAGER_IP="$(tailscale ip --4 manager-node)"
WORKER_TOKEN="<token-from-manager>"

docker swarm join --token $WORKER_TOKEN $SWARM_MANAGER_IP:2377
```

#### 3. Node Labeling

```bash
# Label nodes for specific workloads
docker node update --label-add gpu=true node1
docker node update --label-add gpu=true node2
docker node update --label-add storage=ssd node1
docker node update --label-add storage=hdd node3
docker node update --label-add role=ai-compute node1
docker node update --label-add role=ai-compute node2
docker node update --label-add role=storage node3
docker node update --label-add role=coordinator node4
docker node update --label-add role=monitor node5

# View node labels
docker node ls
docker node inspect node1 --format '{{ .Spec.Labels }}'
```

### ⚙️ Service Deployment Scripts

#### 1. Deploy AI Services Stack

```bash
# /mnt/d/docker-configs/deploy-ai-stack.sh
#!/bin/bash
set -euo pipefail

STACK_NAME="ai-development"
COMPOSE_FILE="/mnt/d/docker-configs/docker-compose.swarm.yml"
LOG_FILE="/mnt/d/logs/deployment.log"

# Create log directory
mkdir -p "$(dirname "$LOG_FILE")"

echo "$(date): Starting AI stack deployment..." | tee -a "$LOG_FILE"

# Verify swarm status
if ! docker info --format '{{.Swarm.LocalNodeState}}' | grep -q active; then
    echo "Error: Docker Swarm is not active" | tee -a "$LOG_FILE"
    exit 1
fi

# Deploy stack
echo "$(date): Deploying stack $STACK_NAME..." | tee -a "$LOG_FILE"
docker stack deploy -c "$COMPOSE_FILE" "$STACK_NAME" 2>&1 | tee -a "$LOG_FILE"

# Wait for services to be ready
echo "$(date): Waiting for services to start..." | tee -a "$LOG_FILE"
sleep 30

# Check service status
echo "$(date): Checking service status..." | tee -a "$LOG_FILE"
docker stack services "$STACK_NAME" | tee -a "$LOG_FILE"

# Health checks
echo "$(date): Performing health checks..." | tee -a "$LOG_FILE"
for service in $(docker stack services --format "{{.Name}}" "$STACK_NAME"); do
    replicas=$(docker service inspect --format '{{.Spec.Mode.Replicated.Replicas}}' "$service")
    ready=$(docker service ls --filter "name=$service" --format "{{.Replicas}}" | cut -d'/' -f1)
    
    if [ "$ready" = "$replicas" ]; then
        echo "✓ $service: $ready/$replicas replicas ready" | tee -a "$LOG_FILE"
    else
        echo "⚠ $service: $ready/$replicas replicas ready" | tee -a "$LOG_FILE"
    fi
done

echo "$(date): AI stack deployment completed" | tee -a "$LOG_FILE"
```

#### 2. Service Management Script

```bash
# /mnt/d/docker-configs/manage-services.sh
#!/bin/bash
set -euo pipefail

STACK_NAME="ai-development"

show_help() {
    cat << EOF
Usage: $0 [COMMAND] [OPTIONS]

Commands:
    deploy      Deploy the AI development stack
    remove      Remove the AI development stack
    update      Update specific service
    scale       Scale service replicas
    logs        Show service logs
    status      Show stack status
    restart     Restart specific service

Examples:
    $0 deploy
    $0 scale ollama-swarm 3
    $0 logs ai-dev-swarm
    $0 update ai-dev-swarm
EOF
}

deploy_stack() {
    echo "Deploying AI development stack..."
    /mnt/d/docker-configs/deploy-ai-stack.sh
}

remove_stack() {
    echo "Removing AI development stack..."
    docker stack rm "$STACK_NAME"
    echo "Waiting for cleanup..."
    sleep 10
}

scale_service() {
    local service_name="$1"
    local replicas="$2"
    
    echo "Scaling $service_name to $replicas replicas..."
    docker service scale "${STACK_NAME}_${service_name}=$replicas"
}

show_logs() {
    local service_name="$1"
    
    echo "Showing logs for $service_name..."
    docker service logs --follow --tail 100 "${STACK_NAME}_${service_name}"
}

update_service() {
    local service_name="$1"
    
    echo "Updating service $service_name..."
    docker service update --force "${STACK_NAME}_${service_name}"
}

show_status() {
    echo "=== Docker Swarm Status ==="
    docker node ls
    echo
    echo "=== Stack Services ==="
    docker stack services "$STACK_NAME"
    echo
    echo "=== Service Tasks ==="
    docker stack ps "$STACK_NAME"
}

restart_service() {
    local service_name="$1"
    
    echo "Restarting service $service_name..."
    docker service update --force --update-parallelism 1 "${STACK_NAME}_${service_name}"
}

case "${1:-help}" in
    deploy)
        deploy_stack
        ;;
    remove)
        remove_stack
        ;;
    scale)
        if [ $# -ne 3 ]; then
            echo "Usage: $0 scale <service_name> <replicas>"
            exit 1
        fi
        scale_service "$2" "$3"
        ;;
    logs)
        if [ $# -ne 2 ]; then
            echo "Usage: $0 logs <service_name>"
            exit 1
        fi
        show_logs "$2"
        ;;
    update)
        if [ $# -ne 2 ]; then
            echo "Usage: $0 update <service_name>"
            exit 1
        fi
        update_service "$2"
        ;;
    status)
        show_status
        ;;
    restart)
        if [ $# -ne 2 ]; then
            echo "Usage: $0 restart <service_name>"
            exit 1
        fi
        restart_service "$2"
        ;;
    help|*)
        show_help
        ;;
esac
```

#### 3. Auto-scaling Configuration

```bash
# /mnt/d/docker-configs/autoscale-config.sh
#!/bin/bash
# Auto-scaling configuration for AI services

set -euo pipefail

STACK_NAME="ai-development"
METRICS_INTERVAL=30
LOG_FILE="/mnt/d/logs/autoscale.log"

# Scaling thresholds
CPU_SCALE_UP_THRESHOLD=80
CPU_SCALE_DOWN_THRESHOLD=30
MEMORY_SCALE_UP_THRESHOLD=85
MEMORY_SCALE_DOWN_THRESHOLD=40
MIN_REPLICAS=1
MAX_REPLICAS=5

log_message() {
    echo "$(date): $1" | tee -a "$LOG_FILE"
}

get_service_metrics() {
    local service_name="$1"
    
    # Get service tasks
    tasks=$(docker service ps --filter "desired-state=running" --format "{{.Node}}" "${STACK_NAME}_${service_name}")
    
    # Calculate average CPU and memory usage across tasks
    total_cpu=0
    total_memory=0
    task_count=0
    
    for task in $tasks; do
        # This would need proper metrics collection (Prometheus, etc.)
        # For now, using docker stats as example
        cpu=$(docker stats --no-stream --format "table {{.CPUPerc}}" 2>/dev/null | grep -v "CPU%" | head -1 | sed 's/%//')
        memory=$(docker stats --no-stream --format "table {{.MemPerc}}" 2>/dev/null | grep -v "MEM%" | head -1 | sed 's/%//')
        
        if [[ "$cpu" =~ ^[0-9]+\.?[0-9]*$ ]] && [[ "$memory" =~ ^[0-9]+\.?[0-9]*$ ]]; then
            total_cpu=$(echo "$total_cpu + $cpu" | bc)
            total_memory=$(echo "$total_memory + $memory" | bc)
            ((task_count++))
        fi
    done
    
    if [ $task_count -gt 0 ]; then
        avg_cpu=$(echo "scale=2; $total_cpu / $task_count" | bc)
        avg_memory=$(echo "scale=2; $total_memory / $task_count" | bc)
        echo "$avg_cpu $avg_memory"
    else
        echo "0 0"
    fi
}

scale_service_if_needed() {
    local service_name="$1"
    local current_replicas=$(docker service inspect --format '{{.Spec.Mode.Replicated.Replicas}}' "${STACK_NAME}_${service_name}")
    
    metrics=$(get_service_metrics "$service_name")
    cpu_usage=$(echo "$metrics" | cut -d' ' -f1)
    memory_usage=$(echo "$metrics" | cut -d' ' -f2)
    
    cpu_int=$(echo "$cpu_usage" | cut -d'.' -f1)
    memory_int=$(echo "$memory_usage" | cut -d'.' -f1)
    
    log_message "Service: $service_name, Replicas: $current_replicas, CPU: ${cpu_usage}%, Memory: ${memory_usage}%"
    
    # Scale up conditions
    if ([ "$cpu_int" -gt "$CPU_SCALE_UP_THRESHOLD" ] || [ "$memory_int" -gt "$MEMORY_SCALE_UP_THRESHOLD" ]) && [ "$current_replicas" -lt "$MAX_REPLICAS" ]; then
        new_replicas=$((current_replicas + 1))
        log_message "Scaling UP $service_name from $current_replicas to $new_replicas replicas"
        docker service scale "${STACK_NAME}_${service_name}=$new_replicas"
    
    # Scale down conditions
    elif [ "$cpu_int" -lt "$CPU_SCALE_DOWN_THRESHOLD" ] && [ "$memory_int" -lt "$MEMORY_SCALE_DOWN_THRESHOLD" ] && [ "$current_replicas" -gt "$MIN_REPLICAS" ]; then
        new_replicas=$((current_replicas - 1))
        log_message "Scaling DOWN $service_name from $current_replicas to $new_replicas replicas"
        docker service scale "${STACK_NAME}_${service_name}=$new_replicas"
    fi
}

# Main monitoring loop
monitor_and_scale() {
    log_message "Starting auto-scaling monitor..."
    
    while true; do
        # Monitor key services
        for service in "ai-dev-swarm" "ollama-swarm"; do
            if docker service inspect "${STACK_NAME}_${service}" >/dev/null 2>&1; then
                scale_service_if_needed "$service"
            fi
        done
        
        sleep "$METRICS_INTERVAL"
    done
}

case "${1:-monitor}" in
    monitor)
        monitor_and_scale
        ;;
    check)
        for service in "ai-dev-swarm" "ollama-swarm"; do
            if docker service inspect "${STACK_NAME}_${service}" >/dev/null 2>&1; then
                metrics=$(get_service_metrics "$service")
                cpu=$(echo "$metrics" | cut -d' ' -f1)
                memory=$(echo "$metrics" | cut -d' ' -f2)
                replicas=$(docker service inspect --format '{{.Spec.Mode.Replicated.Replicas}}' "${STACK_NAME}_${service}")
                echo "$service: $replicas replicas, CPU: ${cpu}%, Memory: ${memory}%"
            fi
        done
        ;;
    *)
        echo "Usage: $0 {monitor|check}"
        echo "  monitor - Start continuous monitoring and auto-scaling"
        echo "  check   - Check current metrics for all services"
        ;;
esac
```

chmod +x /mnt/d/docker-configs/manage-services.sh
chmod +x /mnt/d/docker-configs/autoscale-config.sh
```

[⬆️ Back to TOC](#-table-of-contents)

---

## 💾 Volume Management & Storage

### 🗂️ Storage Architecture

#### 1. Volume Creation Scripts

```bash
# /mnt/d/docker-configs/setup-volumes.sh
#!/bin/bash
set -euo pipefail

echo "Setting up Docker Swarm volumes and storage..."

# Create directory structure
create_directories() {
    echo "Creating directory structure..."
    
    # D: drive (4TB SSD) - High-performance storage
    mkdir -p /mnt/d/{
        fabric/{patterns,sessions,models,backups},
        ollama/{config,logs,backups},
        python-projects/{shared-libs,ai-tools,notebooks,environments},
        vscode/{settings,extensions,workspaces},
        git-repos/{fabric-patterns,ai-models,shared-configs},
        docker-configs/{compose-files,dockerfiles,secrets},
        logs/{swarm,services,applications,monitoring},
        ssl-certs,
        monitoring/{prometheus,grafana,alertmanager}
    }
    
    # E: drive (8TB SATA) - Data storage
    mkdir -p /mnt/e/{
        ollama-data/{models,cache,temp},
        docker-volumes/{postgres,redis,elasticsearch,prometheus-data},
        datasets/{training,validation,inference,processed},
        model-artifacts/{checkpoints,exports,fine-tuned},
        backups/{daily,weekly,monthly},
        shared-workspace/{projects,notebooks,experiments}
    }
    
    # Set permissions
    sudo chown -R $USER:docker /mnt/d /mnt/e
    chmod -R 755 /mnt/d /mnt/e
    
    # Specific permissions for sensitive directories
    chmod 700 /mnt/d/ssl-certs
    chmod 700 /mnt/d/docker-configs/secrets
}

# Create Docker volumes
create_docker_volumes() {
    echo "Creating Docker volumes..."
    
    # Named volumes for Docker Swarm
    docker volume create --driver local \
        --opt type=none \
        --opt o=bind \
        --opt device=/mnt/d/fabric/patterns \
        fabric-patterns
    
    docker volume create --driver local \
        --opt type=none \
        --opt o=bind \
        --opt device=/mnt/e/ollama-data/models \
        ollama-models
    
    docker volume create --driver local \
        --opt type=none \
        --opt o=bind \
        --opt device=/mnt/d/python-projects \
        python-projects
    
    docker volume create --driver local \
        --opt type=none \
        --opt o=bind \
        --opt device=/mnt/e/shared-workspace \
        shared-workspace
    
    docker volume create --driver local \
        --opt type=none \
        --opt o=bind \
        --opt device=/mnt/d/logs \
        swarm-logs
    
    # Synology NFS volumes (if available)
    if ping -c 1 synology-ds920.local >/dev/null 2>&1; then
        docker volume create --driver local \
            --opt type=nfs \
            --opt o=addr=synology-ds920.local,rw,nfsvers=4 \
            --opt device=:/volume1/docker-shared \
            synology-shared
        
        docker volume create --driver local \
            --opt type=nfs \
            --opt o=addr=synology-ds920.local,ro,nfsvers=4 \
            --opt device=:/volume1/ai-models \
            synology-models
    fi
}

# Setup backup directories
setup_backups() {
    echo "Setting up backup structure..."
    
    # Create backup scripts directory
    mkdir -p /mnt/d/backups/scripts
    
    # Fabric patterns backup script
    tee /mnt/d/backups/scripts/backup-fabric.sh > /dev/null <<'EOF'
#!/bin/bash
BACKUP_DATE=$(date +%Y%m%d-%H%M%S)
BACKUP_DIR="/mnt/e/backups/fabric"
SOURCE_DIR="/mnt/d/fabric"

mkdir -p "$BACKUP_DIR"

# Create compressed backup
tar -czf "$BACKUP_DIR/fabric-backup-$BACKUP_DATE.tar.gz" -C "$(dirname "$SOURCE_DIR")" "$(basename "$SOURCE_DIR")"

# Keep only last 30 days of backups
find "$BACKUP_DIR" -name "fabric-backup-*.tar.gz" -mtime +30 -delete

echo "$(date): Fabric backup completed - fabric-backup-$BACKUP_DATE.tar.gz"
EOF

    # Ollama models backup script
    tee /mnt/d/backups/scripts/backup-ollama.sh > /dev/null <<'EOF'
#!/bin/bash
BACKUP_DATE=$(date +%Y%m%d-%H%M%S)
BACKUP_DIR="/mnt/e/backups/ollama"
SOURCE_DIR="/mnt/e/ollama-data"

mkdir -p "$BACKUP_DIR"

# Backup only configuration and custom models
rsync -av --exclude="cache/*" "$SOURCE_DIR/" "$BACKUP_DIR/ollama-backup-$BACKUP_DATE/"

# Create manifest of current models
docker exec ollama-server ollama list > "$BACKUP_DIR/ollama-models-$BACKUP_DATE.txt" 2>/dev/null || true

echo "$(date): Ollama backup completed - ollama-backup-$BACKUP_DATE"
EOF

    # Python projects backup script
    tee /mnt/d/backups/scripts/backup-python.sh > /dev/null <<'EOF'
#!/bin/bash
BACKUP_DATE=$(date +%Y%m%d-%H%M%S)
BACKUP_DIR="/mnt/e/backups/python-projects"
SOURCE_DIR="/mnt/d/python-projects"

mkdir -p "$BACKUP_DIR"

# Backup excluding virtual environments and cache
tar -czf "$BACKUP_DIR/python-projects-$BACKUP_DATE.tar.gz" \
    --exclude="*/venv/*" \
    --exclude="*/.venv/*" \
    --exclude="*/__pycache__/*" \
    --exclude="*/node_modules/*" \
    -C "$(dirname "$SOURCE_DIR")" "$(basename "$SOURCE_DIR")"

echo "$(date): Python projects backup completed - python-projects-$BACKUP_DATE.tar.gz"
EOF

    chmod +x /mnt/d/backups/scripts/*.sh
    
    # Schedule backups
    (crontab -l 2>/dev/null; echo "0 1 * * * /mnt/d/backups/scripts/backup-fabric.sh") | crontab -
    (crontab -l 2>/dev/null; echo "0 2 * * * /mnt/d/backups/scripts/backup-python.sh") | crontab -
    (crontab -l 2>/dev/null; echo "0 3 * * 0 /mnt/d/backups/scripts/backup-ollama.sh") | crontab -
}

# Setup Synology integration
setup_synology() {
    if ! ping -c 1 synology-ds920.local >/dev/null 2>&1; then
        echo "Synology DS920+ not reachable, skipping NFS setup"
        return
    fi
    
    echo "Setting up Synology DS920+ integration..."
    
    # Install NFS utilities
    sudo apt-get update && sudo apt-get install -y nfs-common
    
    # Create mount points
    sudo mkdir -p /mnt/synology/{shared,models,backups}
    
    # Add to fstab for persistent mounting
    if ! grep -q "synology-ds920.local" /etc/fstab; then
        echo "synology-ds920.local:/volume1/docker-shared /mnt/synology/shared nfs defaults 0 0" | sudo tee -a /etc/fstab
        echo "synology-ds920.local:/volume1/ai-models /mnt/synology/models nfs ro,defaults 0 0" | sudo tee -a /etc/fstab
        echo "synology-ds920.local:/volume1/backups /mnt/synology/backups nfs defaults 0 0" | sudo tee -a /etc/fstab
    fi
    
    # Mount NFS shares
    sudo mount -a
    
    # Test connectivity
    if [ -d "/mnt/synology/shared" ]; then
        echo "✓ Synology shared directory mounted successfully"
    else
        echo "✗ Failed to mount Synology shared directory"
    fi
}

# Cleanup old data
cleanup_old_data() {
    echo "Setting up data cleanup..."
    
    # Create cleanup script
    tee /mnt/d/backups/scripts/cleanup-data.sh > /dev/null <<'EOF'
#!/bin/bash
# Clean up old data and temporary files

# Clean Docker system
docker system prune -f
docker volume prune -f

# Clean Ollama cache (keep last 7 days)
find /mnt/e/ollama-data/cache -type f -mtime +7 -delete

# Clean temporary files
find /mnt/e/datasets/temp -type f -mtime +3 -delete
find /mnt/d/logs -name "*.log" -mtime +30 -delete

# Clean old notebook checkpoints
find /mnt/d/python-projects -name ".ipynb_checkpoints" -type d -exec rm -rf {} + 2>/dev/null

echo "$(date): Cleanup completed"
EOF

    chmod +x /mnt/d/backups/scripts/cleanup-data.sh
    
    # Schedule weekly cleanup
    (crontab -l 2>/dev/null; echo "0 4 * * 1 /mnt/d/backups/scripts/cleanup-data.sh") | crontab -
}

# Main execution
main() {
    create_directories
    create_docker_volumes
    setup_backups
    setup_synology
    cleanup_old_data
    
    echo "✓ Volume management setup completed"
    echo "✓ Directory structure created"
    echo "✓ Docker volumes configured"
    echo "✓ Backup scripts installed"
    echo "✓ Cleanup routines scheduled"
    
    # Display volume information
    echo ""
    echo "=== Docker Volumes ==="
    docker volume ls
    
    echo ""
    echo "=== Storage Usage ==="
    df -h /mnt/d /mnt/e /mnt/synology/* 2>/dev/null
}

main "$@"
```

#### 2. Shared Volume Configuration

```yaml
# /mnt/d/docker-configs/shared-volumes.yml
# Shared volume configuration for Docker Swarm

version: '3.8'

services:
  volume-manager:
    image: alpine:latest
    command: ["tail", "-f", "/dev/null"]
    volumes:
      # Fabric shared volumes
      - type: bind
        source: /mnt/d/fabric/patterns
        target: /shared/fabric/patterns
        bind:
          propagation: shared
      - type: bind
        source: /mnt/d/fabric/sessions
        target: /shared/fabric/sessions
        bind:
          propagation: shared
          
      # Ollama shared volumes  
      - type: bind
        source: /mnt/e/ollama-data/models
        target: /shared/ollama/models
        bind:
          propagation: shared
      - type: bind
        source: /mnt/e/ollama-data/cache
        target: /shared/ollama/cache
        bind:
          propagation: shared
          
      # Python development shared volumes
      - type: bind
        source: /mnt/d/python-projects/shared-libs
        target: /shared/python/libs
        bind:
          propagation: shared
      - type: bind
        source: /mnt/e/shared-workspace
        target: /shared/workspace
        bind:
          propagation: shared
          
      # Git repositories
      - type: bind
        source: /mnt/d/git-repos
        target: /shared/git
        bind:
          propagation: shared
          
      # Synology mounts (if available)
      - type: bind
        source: /mnt/synology/shared
        target: /shared/synology
        bind:
          propagation: shared
          
    deploy:
      mode: global
      placement:
        constraints:
          - node.role==manager
    networks:
      - shared-network

networks:
  shared-network:
    driver: overlay
    external: true
```

[⬆️ Back to TOC](#-table-of-contents)

---

## 📊 Monitoring & Maintenance

### 📈 Monitoring Stack Setup

#### 1. Prometheus Configuration

```yaml
# /mnt/d/monitoring/prometheus/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "/etc/prometheus/rules/*.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'docker-swarm'
    dockerswarm_sd_configs:
      - host: unix:///var/run/docker.sock
        role: tasks
    relabel_configs:
      - source_labels: [__meta_dockerswarm_task_desired_state]
        regex: running
        action: keep
      - source_labels: [__meta_dockerswarm_service_name]
        target_label: service_name
      - source_labels: [__meta_dockerswarm_node_hostname]
        target_label: node_hostname

  - job_name: 'ollama'
    static_configs:
      - targets: ['ollama:11434']
    metrics_path: '/api/metrics'
    scrape_interval: 30s

  - job_name: 'fabric-services'
    static_configs:
      - targets: ['ai-dev-swarm:8000']
    metrics_path: '/metrics'
    scrape_interval: 30s

  - job_name: 'node-exporter'
    static_configs:
      - targets: 
          - 'node1:9100'
          - 'node2:9100'
          - 'node3:9100'
          - 'node4:9100'
          - 'node5:9100'

  - job_name: 'cadvisor'
    static_configs:
      - targets:
          - 'node1:8080'
          - 'node2:8080'
          - 'node3:8080'
          - 'node4:8080'
          - 'node5:8080'
```

#### 2. Monitoring Services Stack

```yaml
# /mnt/d/docker-configs/monitoring-stack.yml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:v2.45.0
    ports:
      - "9090:9090"
    volumes:
      - type: bind
        source: /mnt/d/monitoring/prometheus
        target: /etc/prometheus
      - type: bind
        source: /mnt/e/docker-volumes/prometheus-data
        target: /prometheus
      - type: bind
        source: /var/run/docker.sock
        target: /var/run/docker.sock
        read_only: true
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
      - '--storage.tsdb.retention.time=30d'
      - '--web.enable-lifecycle'
      - '--web.enable-admin-api'
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints:
          - node.role==manager
      resources:
        limits:
          memory: 2G
          cpus: '1.0'
    networks:
      - monitoring

  grafana:
    image: grafana/grafana:10.0.0
    ports:
      - "3001:3000"
    volumes:
      - type: bind
        source: /mnt/e/docker-volumes/grafana-data
        target: /var/lib/grafana
      - type: bind
        source: /mnt/d/monitoring/grafana/provisioning
        target: /etc/grafana/provisioning
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin123
      - GF_USERS_ALLOW_SIGN_UP=false
      - GF_INSTALL_PLUGINS=grafana-piechart-panel,grafana-worldmap-panel
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints:
          - node.role==manager
    networks:
      - monitoring

  alertmanager:
    image: prom/alertmanager:v0.25.0
    ports:
      - "9093:9093"
    volumes:
      - type: bind
        source: /mnt/d/monitoring/alertmanager
        target: /etc/alertmanager
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
      - '--storage.path=/alertmanager'
      - '--web.external-url=http://alertmanager:9093'
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints:
          - node.role==manager
    networks:
      - monitoring

  node-exporter:
    image: prom/node-exporter:v1.6.0
    ports:
      - "9100:9100"
    volumes:
      - type: bind
        source: /proc
        target: /host/proc
        read_only: true
      - type: bind
        source: /sys
        target: /host/sys
        read_only: true
      - type: bind
        source: /
        target: /rootfs
        read_only: true
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($|/)'
    deploy:
      mode: global
    networks:
      - monitoring

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.47.0
    ports:
      - "8080:8080"
    volumes:
      - type: bind
        source: /
        target: /rootfs
        read_only: true
      - type: bind
        source: /var/run
        target: /var/run
        read_only: true
      - type: bind
        source: /sys
        target: /sys
        read_only: true
      - type: bind
        source: /var/lib/docker
        target: /var/lib/docker
        read_only: true
      - type: bind
        source: /dev/disk
        target: /dev/disk
        read_only: true
    deploy:
      mode: global
    networks:
      - monitoring

networks:
  monitoring:
    driver: overlay
    attachable: true
```

#### 3. Health Check Scripts

```bash
# /mnt/d/monitoring/scripts/health-check.sh
#!/bin/bash
set -euo pipefail

STACK_NAME="ai-development"
LOG_FILE="/mnt/d/logs/health-check.log"
ALERT_WEBHOOK_URL="${SLACK_WEBHOOK_URL:-}"

log_message() {
    echo "$(date): $1" | tee -a "$LOG_FILE"
}

send_alert() {
    local message="$1"
    local level="${2:-warning}"
    
    log_message "ALERT [$level]: $message"
    
    if [ -n "$ALERT_WEBHOOK_URL" ]; then
        curl -X POST -H 'Content-type: application/json' \
            --data "{\"text\":\"🚨 Docker Swarm Alert [$level]: $message\"}" \
            "$ALERT_WEBHOOK_URL" 2>/dev/null || true
    fi
}

check_swarm_status() {
    log_message "Checking Docker Swarm status..."
    
    if ! docker info --format '{{.Swarm.LocalNodeState}}' | grep -q active; then
        send_alert "Docker Swarm is not active!" "critical"
        return 1
    fi
    
    # Check node availability
    down_nodes=$(docker node ls --filter "availability=drain" --format "{{.Hostname}}" || echo "")
    if [ -n "$down_nodes" ]; then
        send_alert "Nodes are down or drained: $down_nodes" "warning"
    fi
    
    log_message "✓ Docker Swarm status OK"
}

check_services_health() {
    log_message "Checking service health..."
    
    services=$(docker stack services --format "{{.Name}}" "$STACK_NAME" 2>/dev/null || echo "")
    
    if [ -z "$services" ]; then
        send_alert "No services found in stack $STACK_NAME" "critical"
        return 1
    fi
    
    for service in $services; do
        replicas=$(docker service ls --filter "name=$service" --format "{{.Replicas}}")
        desired=$(echo "$replicas" | cut -d'/' -f2)
        running=$(echo "$replicas" | cut -d'/' -f1)
        
        if [ "$running" != "$desired" ]; then
            send_alert "Service $service: $running/$desired replicas running" "warning"
        else
            log_message "✓ Service $service: $running/$desired replicas OK"
        fi
    done
}

check_ollama_connectivity() {
    log_message "Checking Ollama connectivity..."
    
    # Test local Ollama instance
    if curl -s http://localhost:11434/api/tags >/dev/null 2>&1; then
        log_message "✓ Local Ollama instance responsive"
    else
        send_alert "Local Ollama instance not responding" "warning"
    fi
    
    # Test Swarm Ollama service
    if docker service inspect "${STACK_NAME}_ollama-swarm" >/dev/null 2>&1; then
        if curl -s http://ollama-swarm:11434/api/tags >/dev/null 2>&1; then
            log_message "✓ Swarm Ollama service responsive"
        else
            send_alert "Swarm Ollama service not responding" "warning"
        fi
    fi
}

check_storage_usage() {
    log_message "Checking storage usage..."
    
    # Check D: drive (SSD)
    d_usage=$(df -h /mnt/d | awk 'NR==2 {print $5}' | sed 's/%//')
    if [ "$d_usage" -gt 90 ]; then
        send_alert "D: drive usage is ${d_usage}% (SSD storage critical)" "critical"
    elif [ "$d_usage" -gt 80 ]; then
        send_alert "D: drive usage is ${d_usage}% (SSD storage warning)" "warning"
    fi
    
    # Check E: drive (SATA)
    e_usage=$(df -h /mnt/e | awk 'NR==2 {print $5}' | sed 's/%//')
    if [ "$e_usage" -gt 95 ]; then
        send_alert "E: drive usage is ${e_usage}% (Data storage critical)" "critical"
    elif [ "$e_usage" -gt 85 ]; then
        send_alert "E: drive usage is ${e_usage}% (Data storage warning)" "warning"
    fi
    
    log_message "✓ Storage usage: D: ${d_usage}%, E: ${e_usage}%"
}

check_gpu_status() {
    log_message "Checking GPU status..."
    
    if command -v nvidia-smi >/dev/null 2>&1; then
        gpu_info=$(nvidia-smi --query-gpu=utilization.gpu,memory.used,memory.total,temperature.gpu --format=csv,noheader,nounits 2>/dev/null || echo "")
        
        if [ -n "$gpu_info" ]; then
            while IFS=',' read -r util mem_used mem_total temp; do
                util=$(echo "$util" | xargs)
                temp=$(echo "$temp" | xargs)
                
                if [ "$temp" -gt 85 ]; then
                    send_alert "GPU temperature is ${temp}°C (critical)" "critical"
                elif [ "$temp" -gt 80 ]; then
                    send_alert "GPU temperature is ${temp}°C (warning)" "warning"
                fi
                
                log_message "✓ GPU: ${util}% utilization, ${temp}°C"
            done <<< "$gpu_info"
        else
            send_alert "Unable to get GPU information" "warning"
        fi
    else
        log_message "nvidia-smi not available, skipping GPU check"
    fi
}

check_fabric_patterns() {
    log_message "Checking Fabric patterns..."
    
    patterns_dir="/mnt/d/fabric/patterns"
    if [ -d "$patterns_dir" ]; then
        pattern_count=$(find "$patterns_dir" -name "*.md" | wc -l)
        log_message "✓ Found $pattern_count Fabric patterns"
        
        # Check if patterns are accessible
        if [ "$pattern_count" -eq 0 ]; then
            send_alert "No Fabric patterns found in $patterns_dir" "warning"
        fi
    else
        send_alert "Fabric patterns directory not found: $patterns_dir" "warning"
    fi
}

generate_report() {
    log_message "=== Health Check Report ==="
    log_message "Timestamp: $(date)"
    log_message "Swarm Status: $(docker info --format '{{.Swarm.LocalNodeState}}')"
    log_message "Active Nodes: $(docker node ls --filter "availability=active" --format "{{.Hostname}}" | wc -l)"
    log_message "Running Services: $(docker stack services "$STACK_NAME" --format "{{.Name}}" | wc -l)"
    log_message "Storage D: $(df -h /mnt/d | awk 'NR==2 {print $4}') free"
    log_message "Storage E: $(df -h /mnt/e | awk 'NR==2 {print $4}') free"
    
    if command -v nvidia-smi >/dev/null 2>&1; then
        gpu_count=$(nvidia-smi -L | wc -l)
        log_message "GPU Count: $gpu_count"
    fi
    
    log_message "=== End Health Check Report ==="
}

# Main health check execution
main() {
    mkdir -p "$(dirname "$LOG_FILE")"
    
    log_message "Starting comprehensive health check..."
    
    # Run all health checks
    check_swarm_status
    check_services_health
    check_ollama_connectivity
    check_storage_usage
    check_gpu_status
    check_fabric_patterns
    
    generate_report
    
    log_message "Health check completed"
}

# Command line options
case "${1:-full}" in
    full)
        main
        ;;
    swarm)
        check_swarm_status
        ;;
    services)
        check_services_health
        ;;
    storage)
        check_storage_usage
        ;;
    gpu)
        check_gpu_status
        ;;
    ollama)
        check_ollama_connectivity
        ;;
    *)
        echo "Usage: $0 {full|swarm|services|storage|gpu|ollama}"
        echo "  full     - Run all health checks (default)"
        echo "  swarm    - Check Docker Swarm status only"
        echo "  services - Check service health only"
        echo "  storage  - Check storage usage only"
        echo "  gpu      - Check GPU status only"
        echo "  ollama   - Check Ollama connectivity only"
        ;;
esac
```

#### 4. Automated Maintenance Scripts

```bash
# /mnt/d/monitoring/scripts/maintenance.sh
#!/bin/bash
set -euo pipefail

MAINTENANCE_LOG="/mnt/d/logs/maintenance.log"
STACK_NAME="ai-development"

log_message() {
    echo "$(date): $1" | tee -a "$MAINTENANCE_LOG"
}

# System cleanup and optimization
system_cleanup() {
    log_message "Starting system cleanup..."
    
    # Docker system cleanup
    log_message "Cleaning Docker system..."
    docker system prune -f
    docker image prune -f
    docker volume prune -f
    docker network prune -f
    
    # Clean up old logs
    log_message "Cleaning old logs..."
    find /mnt/d/logs -name "*.log" -mtime +30 -delete
    find /var/log -name "*.log.*" -mtime +30 -delete 2>/dev/null || true
    
    # Clean temporary files
    log_message "Cleaning temporary files..."
    find /tmp -type f -mtime +7 -delete 2>/dev/null || true
    find /mnt/e/ollama-data/cache -type f -mtime +7 -delete 2>/dev/null || true
    
    # Clean Python cache
    log_message "Cleaning Python cache..."
    find /mnt/d/python-projects -name "__pycache__" -type d -exec rm -rf {} + 2>/dev/null || true
    find /mnt/d/python-projects -name "*.pyc" -delete 2>/dev/null || true
    
    log_message "System cleanup completed"
}

# Update all components
update_components() {
    log_message "Starting component updates..."
    
    # Update system packages
    log_message "Updating system packages..."
    sudo apt update && sudo apt upgrade -y
    
    # Update Fabric
    log_message "Updating Fabric..."
    /mnt/d/fabric/bin/update-fabric.sh
    
    # Update Ollama and models
    log_message "Updating Ollama..."
    /mnt/d/ollama/bin/update-ollama.sh
    
    # Update Python packages
    log_message "Updating Python packages..."
    cd /mnt/d/python-projects
    if [ -f "pyproject.toml" ]; then
        uv sync --upgrade
    fi
    
    # Update Docker images
    log_message "Updating Docker images..."
    docker pull nvidia/cuda:13.0-cudnn8-devel-ubuntu24.04
    docker pull ollama/ollama:latest
    docker pull prom/prometheus:latest
    docker pull grafana/grafana:latest
    
    log_message "Component updates completed"
}

# Optimize Docker Swarm
optimize_swarm() {
    log_message "Optimizing Docker Swarm..."
    
    # Rebalance services if needed
    services=$(docker stack services --format "{{.Name}}" "$STACK_NAME")
    for service in $services; do
        # Force update to redistribute tasks
        docker service update --force "$service" >/dev/null 2>&1 || true
    done
    
    # Compact Docker daemon logs
    docker system events --filter type=container --since 1h --until now >/dev/null 2>&1 || true
    
    log_message "Docker Swarm optimization completed"
}

# Backup critical data
backup_critical_data() {
    log_message "Starting critical data backup..."
    
    # Run backup scripts
    /mnt/d/backups/scripts/backup-fabric.sh
    /mnt/d/backups/scripts/backup-python.sh
    
    # Backup Docker Swarm configuration
    BACKUP_DATE=$(date +%Y%m%d-%H%M%S)
    BACKUP_DIR="/mnt/e/backups/swarm-config"
    mkdir -p "$BACKUP_DIR"
    
    # Export Swarm join tokens
    docker swarm join-token worker -q > "$BACKUP_DIR/worker-token-$BACKUP_DATE.txt"
    docker swarm join-token manager -q > "$BACKUP_DIR/manager-token-$BACKUP_DATE.txt"
    
    # Export service configurations
    docker stack config "$STACK_NAME" > "$BACKUP_DIR/stack-config-$BACKUP_DATE.yml" 2>/dev/null || true
    
    # Backup monitoring configuration
    tar -czf "$BACKUP_DIR/monitoring-config-$BACKUP_DATE.tar.gz" -C /mnt/d monitoring/
    
    log_message "Critical data backup completed"
}

# Performance tuning
performance_tuning() {
    log_message "Starting performance tuning..."
    
    # GPU performance optimization
    if command -v nvidia-smi >/dev/null 2>&1; then
        # Set GPU persistence mode
        sudo nvidia-smi -pm 1 2>/dev/null || true
        
        # Set GPU power limit (adjust as needed)
        sudo nvidia-smi -pl 300 2>/dev/null || true
        
        log_message "GPU performance settings applied"
    fi
    
    # Docker daemon optimization
    DOCKER_CONFIG="/etc/docker/daemon.json"
    if [ -f "$DOCKER_CONFIG" ]; then
        # Create optimized Docker daemon configuration
        sudo tee "$DOCKER_CONFIG" > /dev/null <<EOF
{
  "default-runtime": "nvidia",
  "runtimes": {
    "nvidia": {
      "path": "nvidia-container-runtime",
      "runtimeArgs": []
    }
  },
  "default-shm-size": "2G",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "50m",
    "max-file": "3"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ],
  "max-concurrent-downloads": 10,
  "max-concurrent-uploads": 5,
  "max-download-attempts": 5
}
EOF
        
        sudo systemctl reload docker
        log_message "Docker daemon configuration optimized"
    fi
    
    # System performance tuning
    # Increase file descriptor limits
    echo "* soft nofile 65536" | sudo tee -a /etc/security/limits.conf
    echo "* hard nofile 65536" | sudo tee -a /etc/security/limits.conf
    
    # Optimize kernel parameters for containers
    sudo tee /etc/sysctl.d/99-docker.conf > /dev/null <<EOF
# Docker optimization
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
vm.max_map_count = 262144
fs.file-max = 65536
EOF
    
    sudo sysctl --system
    
    log_message "Performance tuning completed"
}

# Security hardening
security_hardening() {
    log_message "Starting security hardening..."
    
    # Update and secure SSH configuration
    if [ -f "/etc/ssh/sshd_config" ]; then
        sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.backup
        
        # Apply secure SSH settings
        sudo sed -i 's/#PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config
        sudo sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
        
        sudo systemctl reload sshd
        log_message "SSH configuration hardened"
    fi
    
    # Configure fail2ban for additional security
    if command -v fail2ban-client >/dev/null 2>&1; then
        sudo systemctl enable fail2ban
        sudo systemctl start fail2ban
        log_message "Fail2ban security enabled"
    fi
    
    # Secure Docker daemon
    sudo chmod 660 /var/run/docker.sock
    
    # Update firewall rules for Tailscale
    if command -v ufw >/dev/null 2>&1; then
        sudo ufw --force reset
        sudo ufw default deny incoming
        sudo ufw default allow outgoing
        
        # Allow Tailscale
        sudo ufw allow in on tailscale0
        
        # Allow Docker Swarm ports
        sudo ufw allow 2377  # Swarm management
        sudo ufw allow 7946  # Network discovery
        sudo ufw allow 4789  # Overlay network
        
        # Allow application ports
        sudo ufw allow 8000   # Main application
        sudo ufw allow 11434  # Ollama
        sudo ufw allow 7860   # Marimo
        
        sudo ufw --force enable
        log_message "Firewall rules configured"
    fi
    
    log_message "Security hardening completed"
}

# Generate maintenance report
generate_maintenance_report() {
    log_message "=== Maintenance Report ==="
    log_message "Maintenance completed at: $(date)"
    
    # System status
    log_message "System uptime: $(uptime -p)"
    log_message "Load average: $(uptime | awk -F'load average:' '{print $2}')"
    
    # Docker status
    log_message "Docker containers: $(docker ps -q | wc -l) running"
    log_message "Docker images: $(docker images -q | wc -l) total"
    log_message "Docker volumes: $(docker volume ls -q | wc -l) total"
    
    # Storage status
    log_message "D: drive usage: $(df -h /mnt/d | awk 'NR==2 {print $5}')"
    log_message "E: drive usage: $(df -h /mnt/e | awk 'NR==2 {print $5}')"
    
    # GPU status
    if command -v nvidia-smi >/dev/null 2>&1; then
        gpu_temp=$(nvidia-smi --query-gpu=temperature.gpu --format=csv,noheader,nounits | head -1)
        gpu_util=$(nvidia-smi --query-gpu=utilization.gpu --format=csv,noheader,nounits | head -1)
        log_message "GPU status: ${gpu_util}% utilization, ${gpu_temp}°C"
    fi
    
    # Service status
    if docker stack ls | grep -q "$STACK_NAME"; then
        services_count=$(docker stack services "$STACK_NAME" --format "{{.Name}}" | wc -l)
        log_message "Stack services: $services_count running"
    fi
    
    log_message "=== End Maintenance Report ==="
}

# Main maintenance execution
main() {
    mkdir -p "$(dirname "$MAINTENANCE_LOG")"
    
    log_message "Starting scheduled maintenance..."
    
    # Run maintenance tasks based on argument
    case "${1:-full}" in
        full)
            system_cleanup
            update_components
            optimize_swarm
            backup_critical_data
            performance_tuning
            security_hardening
            ;;
        cleanup)
            system_cleanup
            ;;
        update)
            update_components
            ;;
        optimize)
            optimize_swarm
            performance_tuning
            ;;
        backup)
            backup_critical_data
            ;;
        security)
            security_hardening
            ;;
        *)
            echo "Usage: $0 {full|cleanup|update|optimize|backup|security}"
            exit 1
            ;;
    esac
    
    generate_maintenance_report
    log_message "Scheduled maintenance completed"
}

main "$@"
```

#### 5. Cron Job Setup

```bash
# /mnt/d/monitoring/scripts/setup-cron-jobs.sh
#!/bin/bash
set -euo pipefail

echo "Setting up cron jobs for monitoring and maintenance..."

# Create cron job entries
CRON_JOBS=$(cat <<'EOF'
# Health checks every 15 minutes
*/15 * * * * /mnt/d/monitoring/scripts/health-check.sh full >> /mnt/d/logs/cron.log 2>&1

# Daily maintenance at 2 AM
0 2 * * * /mnt/d/monitoring/scripts/maintenance.sh cleanup >> /mnt/d/logs/cron.log 2>&1

# Weekly full maintenance on Sundays at 3 AM
0 3 * * 0 /mnt/d/monitoring/scripts/maintenance.sh full >> /mnt/d/logs/cron.log 2>&1

# Update components daily at 1 AM
0 1 * * * /mnt/d/monitoring/scripts/maintenance.sh update >> /mnt/d/logs/cron.log 2>&1

# Backup critical data daily at 4 AM
0 4 * * * /mnt/d/monitoring/scripts/maintenance.sh backup >> /mnt/d/logs/cron.log 2>&1

# Auto-scaling monitoring every 5 minutes
*/5 * * * * /mnt/d/docker-configs/autoscale-config.sh check >> /mnt/d/logs/cron.log 2>&1

# Clean up old log files weekly
0 5 * * 1 find /mnt/d/logs -name "*.log" -mtime +30 -delete

# Restart services if needed (monthly)
0 6 1 * * /mnt/d/docker-configs/manage-services.sh restart ai-dev-swarm >> /mnt/d/logs/cron.log 2>&1
EOF
)

# Install cron jobs
echo "$CRON_JOBS" | crontab -

# Verify cron jobs
echo "Installed cron jobs:"
crontab -l

echo "✓ Cron jobs setup completed"
```

[⬆️ Back to TOC](#-table-of-contents)

---

## 🚨 Troubleshooting

### 🔧 Common Issues and Solutions

#### 1. Docker Swarm Issues

```bash
# /mnt/d/troubleshooting/swarm-diagnostics.sh
#!/bin/bash
set -euo pipefail

echo "=== Docker Swarm Diagnostics ==="

# Check Swarm status
echo "1. Swarm Status:"
docker info --format '{{.Swarm.LocalNodeState}}'
docker info --format '{{.Swarm.ControlAvailable}}'

# Check nodes
echo -e "\n2. Node Status:"
docker node ls

# Check network connectivity
echo -e "\n3. Network Connectivity:"
for node in $(docker node ls --format "{{.Hostname}}"); do
    echo "Testing connectivity to $node..."
    if tailscale ping "$node" &>/dev/null; then
        echo "✓ $node - Tailscale connectivity OK"
    else
        echo "✗ $node - Tailscale connectivity FAILED"
    fi
done

# Check service status
echo -e "\n4. Service Status:"
docker stack services ai-development 2>/dev/null || echo "Stack not deployed"

# Check for common issues
echo -e "\n5. Common Issues Check:"

# Check Docker daemon
if ! docker ps &>/dev/null; then
    echo "✗ Docker daemon not running"
else
    echo "✓ Docker daemon running"
fi

# Check Tailscale
if ! tailscale status &>/dev/null; then
    echo "✗ Tailscale not connected"
else
    echo "✓ Tailscale connected"
fi

# Check GPU access
if command -v nvidia-smi &>/dev/null; then
    if nvidia-smi &>/dev/null; then
        echo "✓ GPU accessible"
    else
        echo "✗ GPU not accessible"
    fi
else
    echo "? nvidia-smi not found"
fi

# Check storage mounts
for mount in /mnt/d /mnt/e /mnt/synology; do
    if mountpoint -q "$mount" 2>/dev/null; then
        echo "✓ $mount mounted"
    else
        echo "✗ $mount not mounted"
    fi
done

echo -e "\n=== End Diagnostics ==="
```

#### 2. Ollama Troubleshooting

```bash
# /mnt/d/troubleshooting/ollama-diagnostics.sh
#!/bin/bash
set -euo pipefail

echo "=== Ollama Diagnostics ==="

# Check Ollama service
echo "1. Ollama Service Status:"
if docker ps | grep -q ollama; then
    echo "✓ Ollama container running"
    docker ps --filter "name=ollama" --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
else
    echo "✗ Ollama container not running"
fi

# Check API connectivity
echo -e "\n2. API Connectivity:"
endpoints=("http://localhost:11434" "http://ollama:11434")
for endpoint in "${endpoints[@]}"; do
    if curl -s "$endpoint/api/tags" &>/dev/null; then
        echo "✓ $endpoint - API responsive"
        model_count=$(curl -s "$endpoint/api/tags" | jq '.models | length' 2>/dev/null || echo "0")
        echo "  Models available: $model_count"
    else
        echo "✗ $endpoint - API not responsive"
    fi
done

# Check GPU utilization
echo -e "\n3. GPU Utilization:"
if command -v nvidia-smi &>/dev/null; then
    echo "GPU status during Ollama operation:"
    nvidia-smi --query-gpu=name,utilization.gpu,memory.used,memory.total,temperature.gpu --format=csv
else
    echo "nvidia-smi not available"
fi

# Check model storage
echo -e "\n4. Model Storage:"
model_dirs=("/mnt/e/ollama-data/models" "/root/.ollama/models")
for dir in "${model_dirs[@]}"; do
    if [ -d "$dir" ]; then
        size=$(du -sh "$dir" 2>/dev/null | cut -f1)
        count=$(find "$dir" -name "*.bin" 2>/dev/null | wc -l)
        echo "✓ $dir - Size: $size, Models: $count"
    else
        echo "✗ $dir - Directory not found"
    fi
done

# Test model inference
echo -e "\n5. Model Inference Test:"
test_model="llama3.1:8b"
if curl -s http://localhost:11434/api/tags | jq -r '.models[].name' | grep -q "$test_model"; then
    echo "Testing inference with $test_model..."
    response=$(curl -s -X POST http://localhost:11434/api/generate \
        -d '{"model":"'$test_model'","prompt":"Hello","stream":false}' \
        --max-time 30 2>/dev/null)
    
    if echo "$response" | jq -e '.response' &>/dev/null; then
        echo "✓ Model inference successful"
    else
        echo "✗ Model inference failed"
        echo "Response: $response"
    fi
else
    echo "✗ Test model $test_model not available"
fi

echo -e "\n=== End Ollama Diagnostics ==="
```

#### 3. Performance Troubleshooting

```bash
# /mnt/d/troubleshooting/performance-diagnostics.sh
#!/bin/bash
set -euo pipefail

echo "=== Performance Diagnostics ==="

# System resources
echo "1. System Resources:"
echo "CPU cores: $(nproc)"
echo "Memory total: $(free -h | awk '/^Mem:/ {print $2}')"
echo "Memory available: $(free -h | awk '/^Mem:/ {print $7}')"
echo "Load average: $(uptime | awk -F'load average:' '{print $2}')"

# Storage performance
echo -e "\n2. Storage Performance:"
echo "D: drive (SSD):"
df -h /mnt/d
echo "E: drive (SATA):"
df -h /mnt/e

# Test disk I/O performance
echo -e "\n3. Disk I/O Performance:"
echo "Testing D: drive write speed..."
dd if=/dev/zero of=/mnt/d/testfile bs=1M count=100 2>&1 | grep -E 'MB/s|GB/s' || echo "Test failed"
rm -f /mnt/d/testfile

echo "Testing E: drive write speed..."
dd if=/dev/zero of=/mnt/e/testfile bs=1M count=100 2>&1 | grep -E 'MB/s|GB/s' || echo "Test failed"
rm -f /mnt/e/testfile

# Docker performance
echo -e "\n4. Docker Performance:"
echo "Container count: $(docker ps -q | wc -l)"
echo "Image count: $(docker images -q | wc -l)"
echo "Volume count: $(docker volume ls -q | wc -l)"

# Check for resource-intensive containers
echo -e "\n5. Resource Usage by Container:"
docker stats --no-stream --format "table {{.Container}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.MemPerc}}" | head -10

# GPU performance
echo -e "\n6. GPU Performance:"
if command -v nvidia-smi &>/dev/null; then
    nvidia-smi --query-gpu=name,driver_version,memory.total,memory.used,utilization.gpu,temperature.gpu --format=csv
    echo -e "\nGPU processes:"
    nvidia-smi pmon -c 1 2>/dev/null || echo "No active GPU processes"
else
    echo "NVIDIA GPU not available"
fi

# Network performance
echo -e "\n7. Network Performance:"
echo "Tailscale status:"
tailscale status --peers=false 2>/dev/null || echo "Tailscale not connected"

echo -e "\nDocker network inspection:"
docker network ls --format "table {{.Name}}\t{{.Driver}}\t{{.Scope}}"

# Service response times
echo -e "\n8. Service Response Times:"
services=("http://localhost:8000/health" "http://localhost:11434/api/tags" "http://localhost:7860")
for service in "${services[@]}"; do
    echo -n "Testing $service: "
    response_time=$(curl -o /dev/null -s -w "%{time_total}" "$service" 2>/dev/null || echo "failed")
    if [[ "$response_time" =~ ^[0-9] ]]; then
        echo "${response_time}s"
    else
        echo "failed"
    fi
done

echo -e "\n=== End Performance Diagnostics ==="
```

#### 4. Quick Fix Scripts

```bash
# /mnt/d/troubleshooting/quick-fixes.sh
#!/bin/bash
set -euo pipefail

show_help() {
    cat << EOF
Quick Fix Scripts for Common Issues

Usage: $0 [COMMAND]

Commands:
    restart-docker     Restart Docker daemon
    restart-swarm      Restart Docker Swarm services
    fix-permissions    Fix file permissions
    restart-ollama     Restart Ollama service
    clean-containers   Remove stopped containers
    fix-gpu           Fix GPU access issues
    reset-network     Reset Docker networks
    fix-mounts        Remount storage drives
    full-restart      Complete system restart sequence

EOF
}

restart_docker() {
    echo "Restarting Docker daemon..."
    sudo systemctl restart docker
    sleep 10
    echo "✓ Docker daemon restarted"
}

restart_swarm() {
    echo "Restarting Docker Swarm services..."
    /mnt/d/docker-configs/manage-services.sh remove
    sleep 30
    /mnt/d/docker-configs/manage-services.sh deploy
    echo "✓ Swarm services restarted"
}

fix_permissions() {
    echo "Fixing file permissions..."
    sudo chown -R $USER:docker /mnt/d /mnt/e
    chmod -R 755 /mnt/d /mnt/e
    chmod +x /mnt/d/*/bin/*.sh
    chmod 700 /mnt/d/ssl-certs
    echo "✓ Permissions fixed"
}

restart_ollama() {
    echo "Restarting Ollama service..."
    docker restart ollama-server 2>/dev/null || echo "Ollama container not found"
    sleep 10
    echo "✓ Ollama service restarted"
}

clean_containers() {
    echo "Cleaning up containers..."
    docker container prune -f
    docker image prune -f
    docker volume prune -f
    docker network prune -f
    echo "✓ Container cleanup completed"
}

fix_gpu() {
    echo "Fixing GPU access..."
    sudo nvidia-ctk runtime configure --runtime=docker
    sudo systemctl restart docker
    
    # Test GPU access
    if docker run --rm --runtime=nvidia --gpus all nvidia/cuda:13.0-base-ubuntu24.04 nvidia-smi &>/dev/null; then
        echo "✓ GPU access restored"
    else
        echo "✗ GPU access still not working"
    fi
}

reset_network() {
    echo "Resetting Docker networks..."
    
    # Stop all services first
    docker stack rm ai-development 2>/dev/null || true
    sleep 30
    
    # Remove custom networks
    docker network prune -f
    
    # Restart Docker
    sudo systemctl restart docker
    sleep 10
    
    # Redeploy services
    /mnt/d/docker-configs/manage-services.sh deploy
    echo "✓ Networks reset and services redeployed"
}

fix_mounts() {
    echo "Fixing storage mounts..."
    
    # Remount drives
    sudo umount /mnt/d /mnt/e 2>/dev/null || true
    sudo mount -a
    
    # Check Synology mounts
    if ping -c 1 synology-ds920.local &>/dev/null; then
        sudo umount /mnt/synology/* 2>/dev/null || true
        sudo mount -t nfs synology-ds920.local:/volume1/docker-shared /mnt/synology/shared
        sudo mount -t nfs synology-ds920.local:/volume1/ai-models /mnt/synology/models
    fi
    
    echo "✓ Storage mounts fixed"
}

full_restart() {
    echo "Performing full restart sequence..."
    
    echo "1. Stopping services..."
    docker stack rm ai-development 2>/dev/null || true
    
    echo "2. Cleaning up..."
    clean_containers
    
    echo "3. Fixing permissions..."
    fix_permissions
    
    echo "4. Fixing mounts..."
    fix_mounts
    
    echo "5. Restarting Docker..."
    restart_docker
    
    echo "6. Fixing GPU access..."
    fix_gpu
    
    echo "7. Redeploying services..."
    sleep 30
    /mnt/d/docker-configs/manage-services.sh deploy
    
    echo "✓ Full restart sequence completed"
}

case "${1:-help}" in
    restart-docker) restart_docker ;;
    restart-swarm) restart_swarm ;;
    fix-permissions) fix_permissions ;;
    restart-ollama) restart_ollama ;;
    clean-containers) clean_containers ;;
    fix-gpu) fix_gpu ;;
    reset-network) reset_network ;;
    fix-mounts) fix_mounts ;;
    full-restart) full_restart ;;
    help|*) show_help ;;
esac
```

#### 5. Log Analysis Tools

```bash
# /mnt/d/troubleshooting/analyze-logs.sh
#!/bin/bash
set -euo pipefail

LOG_DIRS=("/mnt/d/logs" "/var/log" "/mnt/d/fabric/logs" "/mnt/d/ollama/logs")
ANALYSIS_OUTPUT="/mnt/d/logs/log-analysis-$(date +%Y%m%d-%H%M%S).txt"

echo "=== Log Analysis Report ===" > "$ANALYSIS_OUTPUT"
echo "Generated: $(date)" >> "$ANALYSIS_OUTPUT"
echo "" >> "$ANALYSIS_OUTPUT"

# Analyze Docker logs
echo "Docker Service Logs:" >> "$ANALYSIS_OUTPUT"
docker service logs ai-development_ai-dev-swarm --tail 50 2>/dev/null >> "$ANALYSIS_OUTPUT" || echo "Service not found" >> "$ANALYSIS_OUTPUT"
echo "" >> "$ANALYSIS_OUTPUT"

# Analyze system logs for errors
echo "Recent System Errors:" >> "$ANALYSIS_OUTPUT"
sudo journalctl --since "1 hour ago" --priority=err -n 20 >> "$ANALYSIS_OUTPUT" 2>/dev/null
echo "" >> "$ANALYSIS_OUTPUT"

# Analyze application logs
echo "Application Log Summary:" >> "$ANALYSIS_OUTPUT"
for log_dir in "${LOG_DIRS[@]}"; do
    if [ -d "$log_dir" ]; then
        echo "=== $log_dir ===" >> "$ANALYSIS_OUTPUT"
        find "$log_dir" -name "*.log" -mtime -1 -exec tail -5 {} + >> "$ANALYSIS_OUTPUT" 2>/dev/null
        echo "" >> "$ANALYSIS_OUTPUT"
    fi
done

# Check for common error patterns
echo "Common Error Patterns:" >> "$ANALYSIS_OUTPUT"
error_patterns=("error" "failed" "exception" "timeout" "connection refused" "out of memory")
for pattern in "${error_patterns[@]}"; do
    count=$(grep -ri "$pattern" /mnt/d/logs/*.log 2>/dev/null | wc -l)
    echo "$pattern: $count occurrences" >> "$ANALYSIS_OUTPUT"
done

echo "" >> "$ANALYSIS_OUTPUT"
echo "Analysis saved to: $ANALYSIS_OUTPUT"
cat "$ANALYSIS_OUTPUT"
```

[⬆️ Back to TOC](#-table-of-contents)

---

## 📝 Final Setup Checklist

### ✅ Pre-Deployment Checklist

- [ ] Windows 11 Pro with WSL2 Ubuntu 24.04 LTS installed
- [ ] NVIDIA RTX 4090/5090 drivers and CUDA 13.0 installed
- [ ] Docker Desktop with WSL2 integration enabled
- [ ] Tailscale installed and configured on all nodes
- [ ] SSH keys generated and distributed
- [ ] Storage drives (D: 4TB SSD, E: 8TB SATA) properly mounted
- [ ] Synology DS920+ network connectivity verified
- [ ] Python 3.13+ installed with UV package manager
- [ ] Go 1.21+ installed for Fabric compilation
- [ ] Git repositories initialized
- [ ] All required directories created with proper permissions

### 🔧 Installation Checklist

- [ ] Fabric installed and configured
- [ ] Ollama Docker container deployed
- [ ] Msty.app configuration completed
- [ ] VSCode with development extensions installed
- [ ] Marimo notebooks environment setup
- [ ] Docker Swarm cluster initialized (5 nodes)
- [ ] All worker nodes joined to swarm
- [ ] Node labels applied for workload distribution

### 🚀 Deployment Checklist

- [ ] AI development stack deployed to swarm
- [ ] Monitoring stack (Prometheus, Grafana) deployed
- [ ] Health check scripts configured and scheduled
- [ ] Backup routines automated
- [ ] Auto-scaling configuration enabled
- [ ] Security hardening applied
- [ ] Firewall rules configured
- [ ] SSL certificates generated (if needed)

### 🔍 Verification Checklist

- [ ] All services responding to health checks
- [ ] GPU acceleration working in containers
- [ ] Ollama models downloading and running
- [ ] Fabric patterns accessible and functional
- [ ] Volume mounts working across all nodes
- [ ] Tailscale connectivity between all nodes
- [ ] Monitoring dashboards displaying metrics
- [ ] Backup scripts executing successfully
- [ ] Auto-scaling responding to load changes
- [ ] Log aggregation and rotation working

---

## 🎯 Quick Start Commands

### 🚀 Essential Commands Reference

```bash
# Deploy the complete AI development stack
/mnt/d/docker-configs/deploy-ai-stack.sh

# Check system health
/mnt/d/monitoring/scripts/health-check.sh

# View service status
/mnt/d/docker-configs/manage-services.sh status

# Update all components
/mnt/d/monitoring/scripts/maintenance.sh update

# Scale Ollama service
/mnt/d/docker-configs/manage-services.sh scale ollama-swarm 3

# Start Marimo notebooks
/mnt/d/python-projects/bin/start-marimo.sh

# Update Fabric to latest version
/mnt/d/fabric/bin/update-fabric.sh

# Run diagnostics
/mnt/d/troubleshooting/swarm-diagnostics.sh

# Quick fix common issues
/mnt/d/troubleshooting/quick-fixes.sh full-restart
```

### 🌐 Access URLs

| Service | URL | Purpose |
|---------|-----|---------|
| **Main Application** | http://localhost:8000 | AI development interface |
| **Ollama API** | http://localhost:11434 | Ollama model API |
| **Marimo Notebooks** | http://localhost:7860 | Interactive notebooks |
| **Prometheus** | http://localhost:9090 | Metrics and monitoring |
| **Grafana** | http://localhost:3001 | Dashboards and visualization |
| **Msty Interface** | http://localhost:3000 | AI chat interface |
| **Alertmanager** | http://localhost:9093 | Alert management |

### 📁 Important Directories

| Directory | Purpose | Drive |
|-----------|---------|-------|
| `/mnt/d/fabric/` | Fabric patterns and configurations | SSD |
| `/mnt/d/python-projects/` | Python development workspace | SSD |
| `/mnt/d/docker-configs/` | Docker and Swarm configurations | SSD |
| `/mnt/e/ollama-data/` | Ollama models and cache | SATA |
| `/mnt/e/shared-workspace/` | Shared development workspace | SATA |
| `/mnt/synology/shared/` | Network shared storage | NFS |
| `/mnt/d/logs/` | Application and system logs | SSD |
| `/mnt/e/backups/` | Automated backups | SATA |

---

## 📚 Additional Resources

### 🔗 Documentation Links

- [Docker Swarm Documentation](https://docs.docker.com/engine/swarm/)
- [Ollama GitHub Repository](https://github.com/ollama/ollama)
- [Fabric GitHub Repository](https://github.com/danielmiessler/fabric)
- [Tailscale Documentation](https://tailscale.com/docs/)
- [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/)
- [UV Package Manager](https://docs.astral.sh/uv/)
- [Marimo Documentation](https://docs.marimo.io/)

### 🎓 Learning Resources

- [Docker Swarm Tutorial](https://docs.docker.com/engine/swarm/swarm-tutorial/)
- [NVIDIA Docker Guide](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html)
- [Prometheus Monitoring](https://prometheus.io/docs/introduction/overview/)
- [Grafana Dashboards](https://grafana.com/docs/grafana/latest/)

### 🆘 Support Resources

- [Docker Community Forum](https://forums.docker.com/)
- [Ollama Discord](https://discord.gg/ollama)
- [Fabric Community](https://github.com/danielmiessler/fabric/discussions)
- [Tailscale Support](https://tailscale.com/support/)

---

## 🔄 Maintenance Schedule

### 📅 Recommended Maintenance Tasks

| Frequency | Task | Script |
|-----------|------|--------|
| **Every 15 minutes** | Health checks | `/mnt/d/monitoring/scripts/health-check.sh` |
| **Daily 1:00 AM** | Component updates | `/mnt/d/monitoring/scripts/maintenance.sh update` |
| **Daily 2:00 AM** | System cleanup | `/mnt/d/monitoring/scripts/maintenance.sh cleanup` |
| **Daily 3:00 AM** | Ollama model updates | `/mnt/d/ollama/bin/update-ollama.sh` |
| **Daily 4:00 AM** | Data backups | `/mnt/d/backups/scripts/backup-fabric.sh` |
| **Weekly Sunday 3:00 AM** | Full maintenance | `/mnt/d/monitoring/scripts/maintenance.sh full` |
| **Monthly 1st day 6:00 AM** | Service restart | Service rotation and optimization |

### 🔐 Security Updates

- **Weekly**: Security patch updates
- **Monthly**: SSL certificate renewal (if applicable)
- **Quarterly**: Security audit and penetration testing
- **Annually**: Complete security review and hardening

---

## 🎉 Conclusion

This comprehensive guide provides a complete setup for a production-ready Docker Swarm environment optimized for AI development with Fabric, Ollama, and modern development tools. The configuration supports:

### ✨ Key Features Delivered

- **🚀 High Performance**: GPU-accelerated AI workloads with NVIDIA RTX 4090/5090
- **🔄 Auto-Scaling**: Dynamic service scaling based on resource utilization
- **🌐 Zero Trust Network**: Secure Tailscale VPN integration
- **📊 Comprehensive Monitoring**: Prometheus, Grafana, and custom health checks
- **🛡️ Security Hardened**: SSH keys, firewall rules, and encrypted communications
- **💾 Redundant Storage**: Multi-tier storage with automated backups
- **🔧 Self-Healing**: Automated maintenance and error recovery
- **📱 Modern UI**: Msty.app integration and Marimo notebooks

### 🚀 Next Steps

1. **Execute the setup scripts** in the order presented
2. **Verify each component** using the provided diagnostic tools
3. **Customize configurations** based on your specific requirements
4. **Monitor performance** and adjust scaling parameters
5. **Develop AI applications** using the provided development environment

### 📞 Support

If you encounter issues during setup:

1. **Run diagnostics**: Use the troubleshooting scripts provided
2. **Check logs**: Review application and system logs for errors  
3. **Apply quick fixes**: Use the automated fix scripts
4. **Consult documentation**: Reference the provided resource links
5. **Community support**: Engage with the respective project communities

**Happy AI Development! 🚀🤖**

[⬆️ Back to TOC](#-table-of-contents)
