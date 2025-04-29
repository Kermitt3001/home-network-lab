# **GNS3 Server Setup on Ubuntu with Docker & QEMU**

This repository documents the setup of a **GNS3 server** on Ubuntu, including **Docker** and **QEMU** for running network simulations. The setup is performed on **MacOS** with a virtual machine running Ubuntu Server.



## **📌 Table of Contents**

- [1️⃣ System Preparation](#1️⃣-system-preparation)
- [2️⃣ Installing GNS3 Server](#2️⃣-installing-gns3-server)
- [3️⃣ Setting Up QEMU for Mikrotik](#3️⃣-setting-up-qemu-for-mikrotik)
- [4️⃣ Configuring Docker](#4️⃣-configuring-docker)
- [5️⃣ Connecting GNS3 GUI](#5️⃣-connecting-gns3-gui)
- [6️⃣ MikroTik Configuration](#6️⃣-mikrotik-configuration)
- [7️⃣ Configuring SSH Key Authentication](#7️⃣-configuring-ssh-key-authentication)
- [8️⃣ Troubleshooting](#8️⃣-troubleshooting)

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
sudo apt install -y python3 python3-pip git curl net-tools telnet network-manager
```

### **Configure Static IP using Network Manager**

1. Open terminal and run:

   ```sh
   nmcli connection show
   ```

   Identify your active connection (e.g., `ens160`).

2. Set a static IP:

   ```sh
   nmcli connection modify ens160 ipv4.addresses 192.168.20.10/24
   nmcli connection modify ens160 ipv4.gateway 192.168.20.1
   nmcli connection modify ens160 ipv4.dns "8.8.8.8 8.8.4.4"
   nmcli connection modify ens160 ipv4.method manual
   nmcli connection up ens160
   ```

3. Verify the configuration:

   ```sh
   nmcli device show ens160
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
3. **Ensure that the GUI version on Mac matches the server version on Ubuntu.**
4. Set the remote server:
   - **Host:** `192.168.20.10`
   - **Port:** `3080`
   - **Username:** `admin`
   - **Password:** `your_password`
5. Click **Apply** and **Test Connection**.

---

## **6️⃣ MikroTik Configuration**

Below is the complete configuration of a MikroTik C52iG-5HaxD2HaxD router (RouterOS 7.18.2), with MAC addresses anonymized as `AA:AA:AA:AA:AA:AA`. Each section is numbered and explained in detail.

### 6.1 Basic Setup – Identity, Time, and Updates
```rsc
/system identity set name=hAP-ax2-Secure
/system clock set time-zone-name=XYZ
/system package update install
/system routerboard upgrade
```
- **Set device name** to `hAP-ax2-Secure` for easy identification.
- **Configure timezone** for accurate logs (`XYZ`).
- **Install system updates** and **upgrade firmware** on the routerboard.

### 6.2 Bridge and VLAN Interface Configuration
```rsc
/interface bridge
add name=bridge-main vlan-filtering=yes

/interface vlan
add interface=bridge-main name=vlan10-HOME vlan-id=10
add interface=bridge-main name=vlan20-OT vlan-id=20
add interface=bridge-main name=vlan30-GUEST vlan-id=30
```
- **Create** a `bridge-main` with **VLAN filtering** enabled.
- **Define** three VLANs on that bridge:
  - VLAN 10: Home network
  - VLAN 20: OT (IoT) devices
  - VLAN 30: Guest network

### 6.3 Wi‑Fi Datapath and SSID Configuration
```rsc
/interface wifi datapath
add bridge=bridge-main disabled=no name=dp-HOME vlan-id=10
add bridge=bridge-main disabled=no name=dp-OT vlan-id=20
add bridge=bridge-main disabled=no name=dp-GUEST vlan-id=30
```
- **Map** each VLAN to a dedicated Wi‑Fi datapath.

```rsc
/interface wifi security
add name=WPA3-HOME authentication-types=wpa3-psk encryption=ccmp,gcmp
add name=WPA3-IoT authentication-types=wpa2-psk,wpa3-psk encryption=ccmp,gcmp
add name=WPA2-GUEST authentication-types=wpa2-psk encryption=ccmp
```
- **Define** three Wi‑Fi security profiles for respective SSIDs.

```rsc
/interface wifi configuration
add name=Home-WIFI disabled=no ssid="YOUR_NAME" security=WPA3-HOME
add name=Home_OT disabled=no ssid="YOUR_NAME" security=WPA3-IoT
add name=Guest-WIFI disabled=no ssid="YOUR_NAME" security=WPA2-GUEST
```
- **Create** SSIDs bound to these profiles.

```rsc
/interface wifi
set [find default-name=wifi1] mode=ap channel.band=5ghz-ax skip-dfs-channels=all configuration=Home-WIFI datapath=dp-HOME disabled=no
set [find default-name=wifi2] mode=ap channel.band=2ghz-ax skip-dfs-channels=all configuration=Home_OT datapath=dp-OT disabled=no
add name=wifi3 mode=ap master-interface=wifi1 mac-address=AA:AA:AA:AA:AA:AA configuration=Guest-WIFI datapath=dp-GUEST disabled=no
```
- **Assign** physical and virtual AP interfaces.

### 6.4 IP Pools and DHCP Servers
```rsc
/ip pool
add name=dhcp-home ranges=192.168.10.100-192.168.10.200
add name=dhcp-iot ranges=192.168.20.100-192.168.20.200
add name=dhcp-guest ranges=192.168.30.100-192.168.30.200

/ip dhcp-server
add name=dhcp-home interface=vlan10-HOME address-pool=dhcp-home lease-time=12h30m
add name=dhcp-ot interface=vlan20-OT address-pool=dhcp-iot lease-time=1d30m
add name=dhcp-guest interface=vlan30-GUEST address-pool=dhcp-guest lease-time=1h30m

/ip dhcp-server network
add address=192.168.10.0/24 gateway=192.168.10.1 dns-server=192.168.10.1
add address=192.168.20.0/24 gateway=192.168.20.1 dns-server=192.168.20.1
add address=192.168.30.0/24 gateway=192.168.30.1 dns-server=192.168.30.1
```
- **Define** DHCP pools and **enable** servers per VLAN.

### 6.5 Interface Lists
```rsc
/interface list
add name=WAN
add name=LAN

/interface list member
add interface=ether1 list=WAN
add interface=bridge-main list=LAN
```
- **Group** interfaces for firewall/NAT rule targeting.

### 6.6 Bridge Ports and VLAN Memberships
```rsc
/interface bridge port
add bridge=bridge-main interface=ether2 frame-types=admit-only-untagged-and-priority-tagged pvid=10
add bridge=bridge-main interface=ether3 frame-types=admit-only-untagged-and-priority-tagged pvid=20
add bridge=bridge-main interface=ether4 frame-types=admit-only-untagged-and-priority-tagged pvid=30
add bridge=bridge-main interface=ether5 frame-types=admit-only-untagged-and-priority-tagged pvid=10

/interface bridge vlan
add bridge=bridge-main tagged=bridge-main,wifi1 vlan-ids=10 untagged=ether2,ether5
add bridge=bridge-main tagged=bridge-main,wifi2 vlan-ids=20 untagged=ether3
add bridge=bridge-main tagged=bridge-main,wifi1 vlan-ids=30 untagged=ether4
```
- **Assign** ports and Wi‑Fi interfaces to VLANs.

### 6.7 IP Addresses and WAN DHCP Client
```rsc
/ip address
add address=192.168.10.1/24 interface=vlan10-HOME
add address=192.168.20.1/24 interface=vlan20-OT
add address=192.168.30.1/24 interface=vlan30-GUEST

/ip dhcp-client
add interface=ether1 use-peer-ntp=no
```
- **Set** router IPs and enable WAN DHCP.

### 6.8 DNS Configuration
```rsc
/ip dns
set servers=1.1.1.1,1.0.0.1,1.1.1.2,9.9.9.9 allow-remote-requests=yes
```
- **Use** public DNS resolvers and **serve** LAN.

### 6.9 Firewall Filter Rules
```rsc
/ip firewall filter
# Drop DNS from WAN
add chain=input protocol=udp dst-port=53 in-interface-list=WAN action=drop comment="Drop DNS UDP from WAN"
add chain=input protocol=tcp dst-port=53 in-interface-list=WAN action=drop comment="Drop DNS TCP from WAN"

# Allow established/related and ICMP
add chain=input connection-state=established,related action=accept comment="Allow established"
add chain=input connection-state=invalid action=drop comment="Drop invalid"
add chain=input protocol=icmp action=accept comment="Allow ICMP"

# Block all other input
add chain=input action=drop comment="Drop all other input"

# Forwarding rules
add chain=forward src-address=192.168.10.0/24 action=accept comment="Allow Home to anywhere"
add chain=forward src-address=192.168.20.0/24 dst-address=192.168.10.0/24 action=drop comment="Block IoT→Home"
add chain=forward src-address=192.168.30.0/24 dst-address=192.168.0.0/16 action=drop comment="Block Guest→LAN"

# TCP anomaly protection
add chain=forward protocol=tcp psd=21,3s,3,1 action=drop comment="Drop FTP port scanning"
add chain=forward protocol=tcp tcp-flags=fin,!ack action=drop comment="Drop suspicious TCP"
add chain=forward protocol=tcp tcp-flags=fin,syn,rst,ack action=drop comment="Drop XMAS scans"
```
- **Protect** and **segregate** VLAN traffic.

### 6.10 Firewall NAT and Masquerading
```rsc
/ip firewall nat
add chain=srcnat out-interface-list=WAN src-address=192.168.0.0/16 action=masquerade comment="Masquerade LAN to WAN"
```
- **Translate** LAN for Internet access.

### 6.11 Firewall Mangle for PS4 Priority
```rsc
/ip firewall mangle
add chain=prerouting dst-address=192.168.20.200 new-connection-mark=PS4_CONN comment="Mark PS4 connections"
add chain=forward connection-mark=PS4_CONN new-packet-mark=PS4_TRAFFIC comment="Mark PS4 traffic"
```
- **Tag** PS4 traffic for QoS.

### 6.12 Queues – Bandwidth Management
```rsc
/queue simple
add name="Guest-Limit" target=vlan30-GUEST max-limit=10M/10M comment="Limit guest bandwidth"

/queue tree
add name="PS4-Priority" parent=global packet-mark=PS4_TRAFFIC priority=1
add name="Default-QoS" parent=global packet-mark=all
```
- **Limit** guest VLAN and **prioritize** PS4.

### 6.13 Firewall Service-Port and Management
```rsc
/ip firewall service-port
set ftp disabled=yes
set tftp disabled=yes
set h323 disabled=yes
set sip disabled=yes

/ip service
set telnet disabled=yes
set ftp disabled=yes
set www disabled=yes
set api disabled=yes
set ssh address=192.168.10.0/24
set winbox address=192.168.10.0/24
```
- **Disable** unused services and **restrict** management to Home VLAN.

### 6.14 Logging and System Notes
```rsc
/system logging action
set 1 disk-lines-per-file=10000
/system logging
add action=disk topics=firewall,info
/system note set show-at-login=no
```
- **Configure** disk logging and **suppress** login note.

### 6.15 Packet Sniffer
```rsc
/tool sniffer set file-name=client196.pcap filter-interface=bridge-main filter-ip-address=192.168.10.196/32 filter-mac-address=AA:AA:AA:AA:AA:AA/FF:FF:FF:FF:FF:FF
```
- **Capture** traffic for analysis.


```

---

## **7️⃣ Configuring SSH Key Authentication**

To enable passwordless login to the GNS3 server, generate an SSH key and configure authentication.

### **Generate an SSH Key on MacOS**

```sh
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -C "your_email@example.com"
```
- **`-t rsa -b 4096`** → Generates a **RSA 4096-bit key** (secure).
- **`-f ~/.ssh/id_rsa`** → Specifies the key location.
- **`-C "your_email@example.com"`** → Adds a comment (optional).

📌 When prompted for a passphrase, press **Enter** to leave it empty (for automatic login).

### **Copy the Public Key to the Server**

Use `ssh-copy-id` to transfer the key to the GNS3 server:

```sh
ssh-copy-id kermitt@192.168.20.22
```

If `ssh-copy-id` is unavailable, manually append the key:

```sh
cat ~/.ssh/id_rsa.pub | ssh kermitt@192.168.20.22 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

### **Test SSH Login Without Password**

Now you should be able to log in without entering a password:

```sh
ssh kermitt@192.168.20.22
```

If successful, SSH authentication is correctly configured! 🎉

### **Optional: Configure SSH for Easy Access**

To avoid typing the full SSH command each time, configure an alias in **~/.ssh/config**:

```sh
nano ~/.ssh/config
```
Add the following:

```
Host HomeLab
    HostName 192.168.20.22
    User kermitt
    IdentityFile ~/.ssh/id_rsa
```

Now you can simply type:

```sh
ssh HomeLab
```
---

## **8️⃣  Troubleshooting**

### **Can't Connect to Server in GUI**

Check if the port is open:

```sh
netstat -tulnp | grep 3080
```

If GNS3 is not listening, restart:

```sh
sudo systemctl restart gns3server
```

### **GNS3 Server Doesn't Start**

After verifying port availability with `netstat`, check logs:

```sh
sudo journalctl -u gns3server --no-pager -n 50
```

Try restarting:

```sh
sudo systemctl restart gns3server
```

### **Telnet Not Working for Switches**

Ensure `dynamips` is installed:

```sh
sudo apt install -y dynamips
```

### **Checking VLANs on Mikrotik**

To list VLANs:

```sh
/interface bridge vlan print
```

To check VLAN assignments:

```sh
/interface bridge port print
```

To view connected devices' MAC addresses:

```sh
/interface bridge host print
```

### **VLAN Filtering Issue**

If VLANs are not working correctly, ensure that `vlan-filtering=yes` is enabled on the bridge:

```sh
/interface bridge set bridge-vlans vlan-filtering=yes
```

Without this setting, VLANs may not function as expected in newer versions of RouterOS.

---

This documentation will be expanded as needed. **Contributions welcome!**

