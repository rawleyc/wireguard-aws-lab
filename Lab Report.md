# 🚀 Lab Report: Deploying WireGuard VPN on AWS EC2 (Ubuntu)

## 📋 Overview

This lab documents the end-to-end deployment and troubleshooting of a high-performance WireGuard VPN on an AWS EC2 instance running Ubuntu 24.04 / 22.04 LTS.

The original deployment script was designed for Amazon Linux and required manual adaptation for Ubuntu.

- **Source Repository:** https://github.com/pprometey/wireguard_aws  
- **Goal:** Establish a secure UDP tunnel routing all internet traffic from a Windows 11 client through an AWS-backed gateway.

---

## 🏗️ Architecture

| Component | Details |
|------------|----------|
| **Server** | AWS EC2 `t3.micro` (Ubuntu 22.04/24.04 LTS) |
| **Client** | Windows 11 WireGuard Client |
| **VPN Subnet** | `10.50.0.0/24` |
| **Listening Port** | UDP `54321` |

---

Here is a cleaner, more structured, GitHub-ready version of your Implementation Steps section only.

You can replace your current section with this:

## 🛠️ Implementation Steps

The deployment was completed in four structured phases:

---

### Phase 1 — System Preparation (Ubuntu Environment Fix)

The original repository was written for Amazon Linux and relied on `dnf`/`yum`.  
Since the target instance was Ubuntu 22.04/24.04 LTS, dependencies were installed manually.

#### Install Required Packages

```bash
sudo apt update
sudo apt install wireguard iptables -y


This ensures:

WireGuard kernel module support

Userspace tools (wg, wg-quick)

NAT functionality via iptables

Phase 2 — Cryptographic Key Generation & Directory Setup

The automation script failed before generating keys. Server identity was created manually.

Generate Server Keys
sudo mkdir -p /etc/wireguard
cd /etc/wireguard

wg genkey | sudo tee server_private.key
cat server_private.key | wg pubkey | sudo tee server_public.key


🔒 Ensure private key permissions are restricted:

sudo chmod 600 server_private.key

Prepare Client Directory
sudo mkdir -p /etc/wireguard/clients


This directory stores individual client configurations.

Phase 3 — WireGuard Interface Configuration

The main configuration file was created at:

/etc/wireguard/wg0.conf

Server Configuration
[Interface]
Address = 10.50.0.1/24
SaveConfig = true
ListenPort = 54321
PrivateKey = <Server_Private_Key>

# Enable NAT for outbound internet access
PostUp = iptables -A FORWARD -i %i -j ACCEPT; \
         iptables -t nat -A POSTROUTING -o ens5 -j MASQUERADE

PostDown = iptables -D FORWARD -i %i -j ACCEPT; \
           iptables -t nat -D POSTROUTING -o ens5 -j MASQUERADE

Critical Configuration Notes

ens5 is the AWS EC2 WAN interface (verify using ip a)

NAT masquerading is required to route client traffic to the internet

Incorrect interface naming will result in handshake success but no data return

Phase 4 — Enable Kernel IP Forwarding

By default, Ubuntu does not forward packets.
The server must be configured to act as a router.

Temporarily Enable IP Forwarding
sudo sysctl -w net.ipv4.ip_forward=1

Make It Persistent Across Reboots
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf

Phase 5 — Start and Enable WireGuard

Once configuration was complete:

sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0

Verify Tunnel Status
sudo wg


Expected output should show:

Interface wg0

Listening port 54321

Peer handshake timestamps

Data transfer counters increasing
