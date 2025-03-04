# 🔥 **Ubuntu Firewall & Kali Linux in a Homelab – My Setup & Troubleshooting**

## 📌 **Overview**
This is my **step-by-step documentation** of how I set up **Ubuntu Server as a firewall** and **added Kali Linux to my homelab** using **VMware Fusion (Apple Silicon)**. It includes **what worked, what didn’t, and how I fixed it**.  

The goal of this setup was to create an **isolated lab environment** where **all traffic from Kali Linux must pass through the Ubuntu firewall**. I ran into several issues along the way, and this write-up serves as both a reference for future projects and a record of troubleshooting solutions.


## **🚀 Steps**

### **1️⃣ Install Ubuntu Server and Configure the Firewall**
#### **🖥️ Step 1: Deploy Ubuntu Server**
- Created a new **Ubuntu Server VM** in VMware Fusion.
- Assigned **two network adapters**:
  - **WAN** → Connected to the external network (**Bridged Mode**).
  - **LAN** → Connected to the internal lab network (**Private to My Mac**).

#### **🔧 Step 2: Configure Network Interfaces**
- Identified the network interfaces using:
  ```
  ip a
  ```
- Modified the **Netplan configuration** to set a static IP for the LAN interface.
- Applied changes:
  ```
  sudo netplan apply
  ```

#### **📡 Step 3: Enable Packet Forwarding**
- Enabled forwarding in the system config:
  ```
  sudo nano /etc/sysctl.conf
  ```
  - Uncommented or added:
    ```
    net.ipv4.ip_forward=1
    ```
- Applied changes:
  ```
  sudo sysctl -p
  ```

#### **🌐 Step 4: Set Up NAT for Internet Access**
- Configured NAT (Masquerading) rules:
  ```
  sudo iptables -t nat -A POSTROUTING -o <WAN_INTERFACE> -j MASQUERADE
  sudo iptables -A FORWARD -i <LAN_INTERFACE> -o <WAN_INTERFACE> -j ACCEPT
  sudo iptables -A FORWARD -i <WAN_INTERFACE> -o <LAN_INTERFACE> -m state --state RELATED,ESTABLISHED -j ACCEPT
  ```
- Saved firewall rules:
  ```
  sudo netfilter-persistent save
  ```

#### **📶 Step 5: Install and Configure DHCP**
- Installed `dnsmasq`:
  ```
  sudo apt install dnsmasq -y
  ```
- Configured `/etc/dnsmasq.conf` for DHCP:
  ```
  interface=<LAN_INTERFACE>
  dhcp-range=<START_IP>,<END_IP>,<LEASE_TIME>
  ```
- Restarted `dnsmasq`:
  ```
  sudo systemctl restart dnsmasq
  ```

#### **✅ Step 6: Verify the Firewall**
- Ensured the firewall VM could access external networks.
- Checked if lab devices received IPs from the firewall.

### **2️⃣ Add Kali Linux to the Lab**
#### **💻 Step 1: Deploy Kali Linux**
- Created a **Kali Linux VM** in VMware Fusion.
- Assigned **one network adapter** to **Private to My Mac**.

#### **🔍 Step 2: Configure Kali's Network**
- Identified the network interface:
  ```
  ip a
  ```
- Requested an IP from the firewall’s DHCP:
  ```
  sudo dhclient <INTERFACE>
  ```
- Verified assigned IP and default gateway:
  ```
  ip route
  ```

#### **📡 Step 3: Test Connectivity**
- **Pinged the firewall**:
  ```
  ping -c 4 <FIREWALL_LAN_IP>
  ```
- **Tested internet access**:
  ```
  ping -c 4 8.8.8.8
  ```
- **Checked DNS resolution**:
  ```
  ping -c 4 google.com
  ```
- **Ran system updates**:
  ```
  sudo apt update
  ```

## **⚠️ Troubleshooting Challenges & Fixes**
### **❌ Issue 1: No Host-Only Network Option in VMware Fusion (Apple Silicon)**
- **Problem:** VMware Fusion on **M2** lacks a **Host-Only (`vmnet2`)** option.
- **Fix:** Used **"Private to My Mac"**, which provides internal networking but requires manual DHCP and NAT configuration.

### **❌ Issue 2: Missing or Unexpected Network Interfaces**
- **Problem:** Ubuntu listed network interfaces as `ens160` and `ens256` instead of `eth0` and `eth1`.
- **Fix:** Used `ip a` to confirm interface names and adjusted **Netplan** settings accordingly.

### **❌ Issue 3: `dnsmasq` Failed Due to Port Conflict**
- **Problem:** `dnsmasq` failed with:
  ```
  failed to create listening socket for port 53: address already in use
  ```
- **Cause:** `systemd-resolved` was using port 53.
- **Fix:** Disabled `systemd-resolved`:
  ```
  sudo systemctl stop systemd-resolved
  sudo systemctl disable systemd-resolved
  sudo systemctl mask systemd-resolved
  sudo systemctl restart dnsmasq
  ```

### **❌ Issue 4: Kali Linux Got the Wrong IP Range (`192.168.45.x` Instead of `192.168.10.x`)**
- **Problem:** Kali Linux was receiving an IP outside the expected lab subnet.
- **Fix:**  
  - Deleted the existing network config:
    ```
    sudo nmcli connection delete "Wired connection 1"
    ```
  - Manually assigned a static IP:
    ```
    sudo nmcli connection add type ethernet ifname eth0 con-name eth0 \
    ipv4.method manual \
    ipv4.addresses <LAB_VM_IP>/<SUBNET_MASK> \
    ipv4.gateway <FIREWALL_IP> \
    ipv4.dns "<DNS_SERVER_1> <DNS_SERVER_2>"
    ```
  - Restarted the connection:
    ```
    sudo nmcli connection up eth0
    ```

### **❌ Issue 5: `apt update` Failing on Kali Linux**
- **Problem:** Couldn’t reach `http.kali.org`, causing `apt update` to fail.
- **Fix:**  
  - Verified NAT rules using:
    ```
    sudo iptables -t nat -L
    ```
  - Ensured packet forwarding was enabled:
    ```
    sudo sysctl net.ipv4.ip_forward
    ```
  - Manually set DNS:
    ```
    echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
    ```
  
## **📌 Notes**
- **VMware Fusion Network Setup:**
  - Ubuntu **(Firewall)** requires **two adapters** (Bridged + Private).
  - Kali **(Lab VM)** requires **one adapter** (Private).
- **DHCP issues?** Restart `dnsmasq` and check `/etc/dnsmasq.conf`.
- **Ensure firewall rules persist** (`iptables` can reset after reboot).

## **📌 Summary**
This guide serves as my **documentation of setting up an Ubuntu-based firewall and adding Kali Linux to my homelab**. I encountered several networking issues, but after troubleshooting, I now have a **fully functional environment** where **all lab traffic is routed through the firewall**. This setup is ready for **future cybersecurity projects, testing, and monitoring.** 🔥🚀
