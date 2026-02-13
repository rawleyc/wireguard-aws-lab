Lab Report: Deploying a WireGuard VPN on AWS EC2 (Ubuntu)
📋 Overview
This lab documents the end-to-end setup and troubleshooting of a high-performance WireGuard VPN on an Amazon EC2 (Ubuntu 24.04/22.04 LTS) instance. This project involved migrating and fixing a deployment script originally designed for Amazon Linux.

Source Repository: pprometey/wireguard_aws

Goal: Establish a secure UDP tunnel for all internet traffic from a Windows 11 client through an AWS-backed gateway.

🏗️ Architecture
Server: AWS EC2 t3.micro (Ubuntu)

Client: Windows 11 WireGuard Client

VPN Subnet: 10.50.0.0/24

Listening Port: UDP 54321

🛠️ Implementation Steps
1. Environment Correction
The original install.sh from the cloned repository utilized dnf and yum. For Ubuntu, the environment was manually prepared:

Bash
sudo apt update && sudo apt install wireguard iptables -y
2. Manual Key & Directory Management
Since the automated scripts failed to create directories with correct permissions, the following manual steps were taken:

Bash
# Generate server identity
wg genkey | sudo tee /etc/wireguard/server_private.key
cat /etc/wireguard/server_private.key | wg pubkey | sudo tee /etc/wireguard/server_public.key

# Prepare client directory
sudo mkdir -p /etc/wireguard/clients
3. Gateway Configuration (/etc/wireguard/wg0.conf)
We configured the interface to handle traffic routing and NAT. Critical Fix: The WAN interface was identified as ens5 (or eth0) to ensure iptables rules applied to the correct hardware.

Ini, TOML
[Interface]
Address = 10.50.0.1/24
SaveConfig = true
ListenPort = 54321
PrivateKey = <Server_Private_Key>

# NAT Masquerading for Internet access
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -t nat -A POSTROUTING -o ens5 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -t nat -D POSTROUTING -o ens5 -j MASQUERADE
4. Kernel Routing
Enabled IPv4 forwarding to allow the server to act as a router:

Bash
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
⚠️ Challenges & Debugging Log
❌ The "Identity Crisis" (OS Mismatch)
Challenge: The cloned repo scripts used apt commands while running on an instance where the user expected dnf, and vice-versa.

Fix: Abandoned the automated initial.sh and performed a manual install via apt.

❌ The "Silent Failure" (Empty Keys)
Challenge: systemd reported Configuration parsing error.

Investigation: Found that wg0.conf had an empty PrivateKey =  field because the script crashed before writing the variable.

Fix: Manually injected the generated key using nano.

❌ The "One-Way Tunnel" (No Internet)
Challenge: Handshake was successful (188 B received), but no data was returned to the client.

Fix: Identified that IP Forwarding was disabled and the PostUp rules were targeting the wrong interface name.

❌ The "Netflix Wall" (Commercial Blocks)
Challenge: Streaming services (Netflix) detected the connection.

Lesson: AWS IP ranges are flagged as "Data Center" IPs. While the VPN is technically perfect, commercial streaming services actively block these ranges to enforce geo-restrictions.

🏁 Conclusion
The project successfully demonstrated that while WireGuard is simple to configure, cloud-specific hurdles (Security Groups, IP Forwarding, and WAN Interface naming) are the primary points of failure. The deployment is now a fully functional, secure gateway.
