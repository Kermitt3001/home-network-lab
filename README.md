# home-network-lab
This project documents the setup of a secure home network with devices like smartphones, a computer, a PlayStation, and a NAS. It includes network design, VLANs, firewalls, monitoring, and GitHub documentation for version control and future reference.



# **GNS3 Server Setup on Ubuntu with Docker & QEMU**

This repository documents the setup of a **GNS3 server** on Ubuntu, including **Docker** and **QEMU** for running network simulations. The setup is performed on **MacOS** with a virtual machine running Ubuntu Server.

## **📌 Table of Contents**
- [1️⃣ System Preparation](#1️⃣-system-preparation)
- [2️⃣ Installing GNS3 Server](#2️⃣-installing-gns3-server)
- [3️⃣ Setting Up QEMU for Mikrotik](#3️⃣-setting-up-qemu-for-mikrotik)
- [4️⃣ Configuring Docker](#4️⃣-configuring-docker)
- [5️⃣ Connecting GNS3 GUI](#5️⃣-connecting-gns3-gui)
- [6️⃣ Troubleshooting](#6️⃣-troubleshooting)

---

## **1️⃣ System Preparation**
### **Set Up a Virtual Machine**
GNS3 requires a dedicated **Ubuntu Server** virtual machine. You can set up a VM using:
- **VMware Fusion** (Mac)
- **VirtualBox**
- **Proxmox**

Download the latest **Ubuntu Server** ISO from [ubuntu.com](https://ubuntu.com/download/server) and install it in your VM.

### **Configure Network Adapter (Bridged Mode)**
To ensure that your Ubuntu VM gets an IP address accessible from your Mac, set the network adapter to **Bridged Mode** in your virtualization software:
- **VMware Fusion:** Go to **Settings > Network Adapter > Bridged (Autodetect)**
- **VirtualBox:** Go to **Settings > Network > Adapter 1 > Bridged Adapter**

### **Update & Install Required Packages**
```sh
sudo apt update && sudo apt upgrade -y
sudo apt install -y python3 python3-pip git curl net-tools telnet
```

### **Check Network Interface**
Find the network interface used for static IP:
```sh
ip a
```
Example output shows interface `ens160`.

### **Assign a Static IP**
Edit the Netplan configuration (replace `ens160` with your interface):
```sh
sudo nano /etc/netplan/00-installer-config.yaml
```
Example config:
```yaml
network:
  ethernets:
    ens160:
      dhcp4: no
      addresses:
        - 192.168.20.10/24
      gateway4: 192.168.20.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
  version: 2
```
Apply changes:
```sh
sudo netplan apply
```

---

## **2️⃣ Installing GNS3 Server**
```sh
sudo add-apt-repository ppa:gns3/ppa
sudo apt update
sudo apt install -y gns3-server gns3-gui
```
### **Enable & Start GNS3 Server**
```sh
sudo systemctl enable gns3server
sudo systemctl start gns3server
sudo systemctl status gns3server
```
Verify it's running:
```sh
ps aux | grep gns3
```

---

## **3️⃣ Setting Up QEMU for Mikrotik**
### **Install QEMU**
```sh
sudo apt install -y qemu-system-x86 qemu-utils
```
Check installation:
```sh
qemu-system-x86_64 --version
```

### **Configure QEMU in GNS3**
1. Open **GNS3 GUI** → **Preferences**.
2. Go to **QEMU > Qemu VMs**.
3. Click **New** → **Custom QEMU VM**.
4. Set `qemu-system-x86_64` as the executable.
5. Assign at least **256MB RAM**.
6. Select **e1000** as the network adapter.

---

## **4️⃣ Configuring Docker**
### **Install Docker**
```sh
sudo apt install -y docker.io
sudo systemctl enable --now docker
```
Add user to the `docker` group:
```sh
sudo usermod -aG docker $USER
newgrp docker
```
### **Test Docker**
```sh
docker run hello-world
```
### **Run GNS3 in Docker**
```sh
docker run -d --name gns3server -p 3080:3080 gns3/gns3server
```
Check logs:
```sh
docker logs gns3server
```

---

## **5️⃣ Connecting GNS3 GUI**
1. Open **GNS3 GUI** on **Mac/Windows**.
2. Go to **Preferences > Server**.
3. Set the remote server:
   - **Host:** `192.168.20.10`
   - **Port:** `3080`
   - **Username:** `admin`
   - **Password:** `your_password`
4. Click **Apply** and **Test Connection**.

---

## **6️⃣ Troubleshooting**
### **GNS3 Server Doesn't Start**
Check logs:
```sh
sudo journalctl -u gns3server --no-pager -n 50
```
Try restarting:
```sh
sudo systemctl restart gns3server
```

### **Can't Connect to Server in GUI**
Check if the port is open:
```sh
netstat -tulnp | grep 3080
```
If GNS3 is not listening, restart:
```sh
sudo systemctl restart gns3server
```

### **Telnet Not Working for Switches**
Ensure `dynamips` is installed:
```sh
sudo apt install -y dynamips
```

---

## **💾 Next Steps**
- Upload `.gns3project` files for reusable topologies.
- Automate setup using Ansible or Terraform.

This documentation will be expanded as needed. **Contributions welcome!** 🚀


