# Xilinx Development Environment Setup Guide

Complete setup guide for Ubuntu 22.04.0 with Kernel 5.15, Docker, Vivado 2022.2, and FINN for FPGA development.

---

## Table of Contents

1. [System Requirements](#system-requirements)
2. [Ubuntu 22.04.0 Installation](#ubuntu-22040-installation)
3. [Kernel 5.15 Setup](#kernel-515-setup)
4. [Docker Installation](#docker-installation)
5. [Vivado 2022.2 Installation](#vivado-20222-installation)
6. [FINN Setup](#finn-setup)
7. [Verification](#verification)
8. [Troubleshooting](#troubleshooting)

---

## System Requirements

### Hardware Requirements (VMware Virtual Machine)
- **CPU**: 64-bit x86 processor with virtualization support (Intel VT-x or AMD-V)
  - Recommended: 4 cores assigned to VM
- **RAM**: Minimum 4 GB 
  - 8 GB recommended for better performance with Vivado
- **Storage**: 100 GB virtual disk
  - 60 GB for Ubuntu OS + Vivado
  - 20 GB for Docker containers and FINN
  - 20 GB for workspace and projects
- **VMware Version**: VMware Workstation 15+ or VMware Player 15+
- **GPU** (Optional): GPU passthrough for ML acceleration (advanced setup)

### Software Requirements
- Ubuntu 22.04.1 LTS Desktop ISO
- Vivado 2022.2 installer (from Xilinx website)
- VMware Workstation/Player installed on host machine
- Internet connection for package downloads

### VMware Configuration Notes
- Enable "Virtualize Intel VT-x/EPT or AMD-V/RVI" in VM settings
- Allocate at least 4 CPU cores if host has 8+ cores
- Enable 3D graphics acceleration for better GUI performance
- Use NAT or Bridged network for internet access

---

## Ubuntu 22.04.1 Installation

### Step 1: Download Ubuntu 22.04.1 LTS

Download Ubuntu 22.04.1 Desktop ISO from:
**https://old-releases.ubuntu.com/releases/22.04.1/ubuntu-22.04.1-desktop-amd64.iso**

### Step 4: Post-Installation Updates

```bash
# Update package lists and upgrade system
sudo apt update && sudo apt upgrade -y

# Install essential build tools
sudo apt install -y build-essential git curl wget vim \
    net-tools software-properties-common

# Reboot system
sudo reboot
```

---

## Kernel 5.15 Setup

Ubuntu 22.04.1 comes with **two kernels**: 6.8 (default) and 5.15. By default, the system boots into kernel 6.8, but Vivado 2022.2 works better with kernel 5.15.

### Check Current Kernel

```bash
# Check current kernel version
uname -r
# Output: 6.8.0-xx-generic (default) or 5.15.0-xx-generic (target)

# List all installed kernels
dpkg --list | grep linux-image
# You should see both kernel 6.8 and 5.15
```

### Switch to Kernel 5.15

```bash
# Edit GRUB config
sudo nano /etc/default/grub

# Change: GRUB_DEFAULT=0
# To: GRUB_DEFAULT=1

# This switches from first entry (kernel 6.8) to second entry (kernel 5.15)

# Update GRUB and reboot
sudo update-grub
sudo reboot

# After reboot, verify kernel
uname -r
# Expected: 5.15.0-xx-generic
```

### Lock Kernel Version (Prevent Auto-Updates)

To prevent automatic kernel updates from changing your kernel:

```bash
# Get your current kernel version first
uname -r

# Hold kernel 5.15 packages (replace version number with your actual version)
sudo apt-mark hold linux-image-$(uname -r)
sudo apt-mark hold linux-headers-$(uname -r)
sudo apt-mark hold linux-modules-extra-$(uname -r)

# Verify held packages
apt-mark showhold

# To unhold later (if needed):
# sudo apt-mark unhold linux-image-$(uname -r)
```

### Remove Kernel 6.8 (Optional)

If you don't need kernel 6.8, you can remove it to avoid confusion:

```bash
# List installed kernels
dpkg --list | grep linux-image

# Remove kernel 6.8 (ONLY after confirming 5.15 works!)
sudo apt remove --purge linux-image-6.8.0-*-generic
sudo apt remove --purge linux-headers-6.8.0-*-generic

# Update GRUB
sudo update-grub

# Clean up old packages
sudo apt autoremove
```

**Warning**: Only remove kernel 6.8 after confirming that kernel 5.15 boots successfully and all hardware works correctly.

---

## Docker Installation

### Step 1: Remove Old Docker Versions

```bash
sudo apt remove -y docker docker-engine docker.io containerd runc
```

### Step 2: Install Docker Prerequisites

```bash
sudo apt update
sudo apt install -y \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
```

### Step 3: Add Docker Official GPG Key

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
    sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

### Step 4: Set Up Docker Repository

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### Step 5: Install Docker Engine

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io \
    docker-buildx-plugin docker-compose-plugin
```

### Step 6: Configure Docker Permissions

```bash
# Add current user to docker group
sudo usermod -aG docker $USER

# Apply group changes (or logout/login)
newgrp docker

# Enable Docker to start on boot
sudo systemctl enable docker
sudo systemctl start docker
```

### Step 7: Verify Docker Installation

```bash
# Check Docker version
docker --version
# Expected: Docker version 24.x.x

# Run test container
docker run hello-world

# Check Docker info
docker info
```

### Step 8: Configure Docker Resources (Optional)

Create/edit Docker daemon configuration:

```bash
sudo nano /etc/docker/daemon.json
```

Add resource limits:

```json
{
  "default-runtime": "runc",
  "storage-driver": "overlay2",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

Restart Docker:

```bash
sudo systemctl restart docker
```

---

## Vivado 2022.2 Installation

### Step 1: Download Vivado 2022.2

1. Visit Xilinx Downloads page: https://www.xilinx.com/support/download.html
2. Navigate to "Vivado Archive"
3. Select version **2022.2**
4. Download:
   - **Xilinx Unified Installer 2022.2: Linux Self Extracting Web Installer** (recommended)
   - OR **Full Product Installation** (if offline installation needed)

### Step 2: Install Required Dependencies

```bash
# Install Vivado dependencies
sudo apt update
sudo apt install -y \
    libtinfo5 \
    libncurses5 \
    libncurses5-dev \
    libncursesw5 \
    libncursesw5-dev \
    libtinfo-dev \
    libssl-dev \
    libssl1.1 \
    libc6-i386 \
    lib32z1 \
    lib32ncurses6 \
    zlib1g-dev \
    gcc-multilib \
    g++-multilib
```

### Step 3: Extract and Run Installer

```bash
# Navigate to download directory
cd ~/Downloads

# Make installer executable
chmod +x Xilinx_Unified_2022.2_1014_8888_Lin64.bin

# Run installer
sudo ./Xilinx_Unified_2022.2_1014_8888_Lin64.bin
```

### Step 4: Installation Configuration

1. **Login**: Enter Xilinx account credentials
2. **Select Product**: Choose **Vivado**
3. **Select Edition**: 
   - Vivado ML Enterprise (full features)
   - OR Vivado ML Standard (recommended for most users)
4. **Customize Installation**:
   - Check "Vivado Design Suite"
   - Check "DocNav"
   - Devices: Select your target devices (e.g., Zynq UltraScale+ MPSoC for ZCU102)
5. **Installation Directory**: `/tools/Xilinx` (or your preferred path)
6. **Accept License Agreements**
7. **Install**

Installation takes 1-3 hours depending on internet speed and selected components.

### Step 5: Install Cable Drivers

```bash
# Navigate to Vivado installation
cd /tools/Xilinx/Vivado/2022.2/data/xicom/cable_drivers/lin64/install_script/install_drivers

# Install drivers
sudo ./install_drivers

# Add user to dialout group for USB cable access
sudo usermod -aG dialout $USER
```

### Step 6: Set Up Environment Variables

Add to `~/.bashrc`:

```bash
nano ~/.bashrc
```

Add these lines at the end:

```bash
# Xilinx Vivado 2022.2 Setup
export XILINX_VIVADO=/tools/Xilinx/Vivado/2022.2
export XILINX_HLS=/tools/Xilinx/Vitis_HLS/2022.2
source $XILINX_VIVADO/settings64.sh
```

Apply changes:

```bash
source ~/.bashrc
```

### Step 7: Verify Vivado Installation

```bash
# Check Vivado version
vivado -version
# Expected: Vivado v2022.2

# Launch Vivado GUI (test)
vivado &
```

---

## FINN Setup

FINN (Fast Inference for Neural Networks) is a framework for building FPGA-based neural network accelerators.

### Step 1: Install FINN Prerequisites

```bash
# Install Python 3.8+ and pip
sudo apt update
sudo apt install -y python3 python3-pip python3-venv

# Install Git LFS (for large files)
sudo apt install -y git-lfs
git lfs install

# Install additional dependencies
sudo apt install -y \
    graphviz \
    libgraphviz-dev \
    protobuf-compiler
```

### Step 2: Clone FINN Repository

```bash
# Create workspace directory
mkdir -p ~/workspace/finn
cd ~/workspace/finn

# Clone FINN repository
git clone https://github.com/Xilinx/finn.git
cd finn

# Checkout stable version (or use main branch)
git checkout v0.9  # Use latest stable release
```

### Step 3: Set Up FINN Docker Environment

FINN uses Docker for reproducible builds:

```bash
# Pull FINN Docker image
cd ~/workspace/finn/finn
./run-docker.sh

# This will:
# 1. Build/pull the FINN Docker image
# 2. Mount your local directory
# 3. Start an interactive shell in the container
```

### Step 4: Configure FINN for Vivado Integration

Inside FINN Docker container:

```bash
# Set Vivado environment
export VIVADO_PATH=/tools/Xilinx/Vivado/2022.2
export PLATFORM_REPO_PATHS=/tools/Xilinx/platforms

# Verify Vivado is accessible
which vivado
```

### Step 5: Run FINN Quickstart Example

Inside FINN Docker container:

```python
# Start Python
python3

# Import FINN
from finn.util.test import get_test_model_trained
import finn.builder.build_dataflow as build

# Get example model
model = get_test_model_trained("TFC", 1, 1)

# Build dataflow accelerator (example)
# This is a quick test - real builds take longer
print("FINN is ready!")
```

### Step 6: FINN Environment Setup (Host Machine)

Create a convenience script to enter FINN environment:

```bash
nano ~/finn_env.sh
```

Add:

```bash
#!/bin/bash
# FINN Environment Setup

export FINN_ROOT=~/workspace/finn/finn
export VIVADO_PATH=/tools/Xilinx/Vivado/2022.2
export PLATFORM_REPO_PATHS=/tools/Xilinx/platforms

cd $FINN_ROOT
./run-docker.sh
```

Make executable:

```bash
chmod +x ~/finn_env.sh
```

Usage:

```bash
# Start FINN environment
~/finn_env.sh
```

---

## Verification

### System Verification Checklist

```bash
# 1. Check Ubuntu version
lsb_release -a
# Expected: Ubuntu 22.04.1 LTS

# 2. Check kernel version
uname -r
# Expected: 5.15.0-xx-generic

# 3. Check Docker
docker --version
docker run hello-world

# 4. Check Vivado
source /tools/Xilinx/Vivado/2022.2/settings64.sh
vivado -version

# 5. Check FINN
cd ~/workspace/finn/finn
docker images | grep finn
```

### Create Verification Script

```bash
nano ~/verify_setup.sh
```

Add:

```bash
#!/bin/bash
echo "================================"
echo "Environment Verification Script"
echo "================================"

echo -e "\n1. Ubuntu Version:"
lsb_release -d

echo -e "\n2. Kernel Version:"
uname -r

echo -e "\n3. Docker Version:"
docker --version

echo -e "\n4. Vivado Version:"
source /tools/Xilinx/Vivado/2022.2/settings64.sh
vivado -version 2>/dev/null | head -n 1

echo -e "\n5. FINN Docker Images:"
docker images | grep finn | wc -l
echo "FINN images found: $(docker images | grep finn | wc -l)"

echo -e "\n6. Disk Space:"
df -h / | tail -1

echo -e "\n7. Memory:"
free -h | grep Mem

echo -e "\n================================"
echo "Verification Complete!"
echo "================================"
```

Make executable and run:

```bash
chmod +x ~/verify_setup.sh
~/verify_setup.sh
```

---

## Troubleshooting

### Docker Issues

**Problem**: Permission denied when running Docker

```bash
# Solution: Add user to docker group
sudo usermod -aG docker $USER
newgrp docker
```

**Problem**: Docker daemon not running

```bash
# Solution: Start Docker service
sudo systemctl start docker
sudo systemctl enable docker
```

### Vivado Issues

**Problem**: `libncurses.so.5` not found

```bash
# Solution: Install ncurses5 compatibility
sudo apt install -y libncurses5 libtinfo5
```

**Problem**: Cable drivers not working

```bash
# Solution: Reinstall drivers and add user to dialout
cd /tools/Xilinx/Vivado/2022.2/data/xicom/cable_drivers/lin64/install_script/install_drivers
sudo ./install_drivers
sudo usermod -aG dialout $USER
# Logout and login again
```

**Problem**: License issues

```bash
# Solution: Set license file path
export XILINXD_LICENSE_FILE=/path/to/Xilinx.lic
# Add to ~/.bashrc for persistence
```

### FINN Issues

**Problem**: FINN Docker fails to build

```bash
# Solution: Pull pre-built image
docker pull xilinx/finn:v0.9

# Or build with more memory
docker build --memory=8g -t finn .
```

**Problem**: Vivado not found in FINN container

```bash
# Solution: Mount Vivado directory when running FINN
./run-docker.sh -v /tools/Xilinx:/tools/Xilinx
```

### Kernel Issues

**Problem**: Wrong kernel version after update

```bash
# Solution: Boot into correct kernel via GRUB
# At boot, select Advanced Options > Select 5.15.0-xx kernel

# Set as default:
sudo nano /etc/default/grub
# Set: GRUB_DEFAULT="Advanced options for Ubuntu>Ubuntu, with Linux 5.15.0-xx-generic"
sudo update-grub
```

---

## Additional Resources

### Documentation Links

- **Ubuntu 22.04**: https://help.ubuntu.com/22.04/
- **Docker**: https://docs.docker.com/
- **Vivado 2022.2**: https://docs.xilinx.com/v/u/2022.2-English
- **FINN**: https://finn.readthedocs.io/
- **Xilinx Forums**: https://support.xilinx.com/

### Recommended Tools

```bash
# Install useful development tools
sudo apt install -y \
    gtkwave \           # Waveform viewer
    verilator \         # Verilog simulator
    python3-numpy \     # Python numerical computing
    python3-matplotlib  # Python plotting
```

### Performance Optimization

```bash
# Increase swap space for large builds
sudo fallocate -l 32G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

---
