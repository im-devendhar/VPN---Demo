
# How to create a VPN for secure remote access and how to allow VPN clients to reach resources inside an AWS VPC (EC2 instances). It includes two common approaches you can choose from:

* **Managed (AWS Client VPN)** — simpler, AWS-managed service, recommended for production/AWS-integrated environments.
* **Self-managed (OpenVPN / WireGuard on EC2)** — more control, cheaper but needs more ops work.

At the end you’ll find example commands, troubleshooting tips, and best practices. Use this as a ready-to-drop README for your repo.

---

## Table of contents

1. Overview
2. Prerequisites
3. Option A — AWS Client VPN (managed)

   * Architecture
   * Step-by-step
4. Option B — OpenVPN Access Server on EC2 (self-managed)

   * Architecture
   * Step-by-step
5. Option C — WireGuard on EC2 (lightweight/self-managed)
6. Allowing VPN clients to access EC2 instances (routing & security groups)
7. Example commands & snippets
8. Troubleshooting
9. Security & best practices
10. Cleanup

---

## 1) Overview

A VPN creates an encrypted tunnel so remote users (laptops, home PCs) can securely access private resources inside your VPC (for example, EC2 instances that are not publicly exposed).

**High-level steps common to all approaches:**

1. Create VPN server (managed service or server on EC2).
2. Configure authentication and authorization (certs, Active Directory, or client certs).
3. Configure the VPN’s client CIDR (IP range assigned to connected clients).
4. Ensure the VPC route table knows how to route traffic to the VPN (associate subnets or add routes).
5. Adjust Security Groups and NACLs to permit VPN client traffic to EC2 targets.
6. Provide client config/credentials to users.

---

## 2) Prerequisites

* AWS account with permissions to create: VPC, subnets, EC2, Security Groups, IAM, ACM, Client VPN endpoint, CloudWatch (optional).
* Basic Linux/ssh familiarity for the self-managed options.
* Domain name (optional) if you want certificate-based setup and friendly endpoint name.
* For AWS Client VPN: ACM certificate in the same region (or import into ACM).

---

## 3) Option A — AWS Client VPN (Managed)

### Architecture (simple)

```
Remote Client  <----TLS---->  AWS Client VPN Endpoint  <----VPC routes---->  Private Subnet(s)  --> EC2 instances
```

### When to choose this

* You want minimal ops and good AWS integration.
* You need Active Directory or SAML authentication options.

### Step-by-step

1. **Create / import a server certificate in ACM**

   * Use ACM private CA or import your server certificate (must be in the certificate store of the region where endpoint will be created).
   * For certificate-based auth you need a server cert + client certs (if using mutual TLS). For user-based auth you can use Active Directory or SAML.

2. **Create a Client VPN endpoint**

   * Console: VPC > Client VPN Endpoints > Create Client VPN Endpoint.
   * Required fields: Client IPv4 CIDR (e.g., `10.100.0.0/22`), Server certificate ARN (ACM), Authentication options (mutual auth with certs OR user-based such as Active Directory or Federated auth).

3. **Authorize networks to access**

   * Create authorization rules (who can access what). For example, allow `0.0.0.0/0` to access the VPC (or better, restrict to only subnets/targets).

4. **Associate target networks (subnets)**

   * Associate at least one subnet in each AZ you want clients to reach. This creates elastic network interfaces (ENIs) in the subnets.

5. **Add routes**

   * Add route entries to the Client VPN endpoint: route to your VPC CIDR (e.g., `10.0.0.0/16`) via the associated subnets.

6. **Adjust Security Groups**

   * The ENIs created for the Client VPN will use a security group; ensure that this SG allows inbound traffic from the client CIDR to destination ports or allows outbound to the VPC.
   * Ensure EC2 instance security groups allow inbound traffic from the VPN client CIDR (e.g., `10.100.0.0/22`).

7. **Download client configuration**

   * From the Console you can download the `.ovpn` style client configuration (or provide SAML instructions). Clients use the AWS VPN client or OpenVPN-compatible clients.

8. **Test**

   * Connect from a client, verify you get an IP from the client CIDR, and ping an EC2 private IP.

**Notes:**

* Managed, autoscaled, supported by AWS. No need to manage a server or patch OS.
* Integration with Active Directory, SAML, certificate-based auth possible.

---

## 4) Option B — OpenVPN Access Server on EC2 (Self-managed)

### Architecture (simple)

```
Remote Client <--TLS(OpenVPN)--> EC2 (OpenVPN AS) <--Private network--> EC2 app servers
```

### When to choose this

* Need complete control or want a self-hosted OpenVPN solution.
* You’re comfortable managing EC2, OS updates, scaling, certificates.

### Step-by-step (quick install using OpenVPN Access Server AMI or community OpenVPN)

**A. Provision EC2 instance**

* Recommended: t3.micro/t3.small (small proof-of-concept), use Amazon Linux 2 or Ubuntu.
* Put the instance in a **public subnet** with an Elastic IP (EIP) if you want users to connect from the internet.
* Create a Security Group for the VPN server that allows:

  * UDP 1194 (default OpenVPN) or TCP 443 if you prefer SSL over 443
  * SSH 22 (your admin IP only)
  * ICMP (optional for diagnostics)

**B. Install OpenVPN (server)**
Use the community OpenVPN install script (example for Ubuntu) or use OpenVPN Access Server marketplace AMI.

*Community OpenVPN example (Ubuntu):*

```bash
sudo apt update && sudo apt install -y openvpn easy-rsa
make-cadir ~/openvpn-ca
cd ~/openvpn-ca
# edit vars, then build CA and server certs
./easyrsa init-pki
./easyrsa build-ca nopass
./easyrsa gen-req server nopass
./easyrsa sign-req server server
./easyrsa gen-dh
openvpn --genkey --secret ta.key
# copy server certs to /etc/openvpn
```

Or use Access Server AMI which has UI to manage users and config.

**C. Enable IP forwarding & NAT**
On the OpenVPN server enable forwarding in `/etc/sysctl.conf`:

```
net.ipv4.ip_forward=1
```

Reload: `sudo sysctl -p`

Set NAT rules so VPN clients can reach private subnet instances (assuming `tun0` is VPN interface and VPC CIDR is `10.0.0.0/16`):

```bash
sudo iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE
# Save iptables rules (varies by distro)
```

**D. Client config**

* Generate `.ovpn` files for each user (contains server info and client certs) or use Access Server UI.
* Distribute securely.

**E. Routing and Security**

* Add route on the VPC route table if you used a secondary network or if you are using VPN instance as a router (often not necessary if you NAT).
* Update security groups of EC2 targets to allow inbound from VPN client CIDR (e.g., `10.8.0.0/24`).

**F. Test**

* Connect a client, check `ifconfig`/`ip addr` for assigned VPN IP and `ping` private EC2 IPs.

**Maintenance/Scaling:**

* Use Auto Scaling + internal load balancer for High Availability (more advanced: use multiple OpenVPN servers with a shared authentication backend and sticky client routing).
* Regularly update the EC2 instance OS and OpenVPN packages.

---

## 5) Option C — WireGuard on EC2 (Lightweight Self-managed)

WireGuard is modern, faster, and easier to configure than classic OpenVPN. Good for small teams and when you want a minimal stack.

**Quick steps**

1. Launch EC2 in public subnet with EIP and necessary SG (UDP port you choose, e.g., 51820).
2. Install wireguard: `sudo yum install -y wireguard-tools` (Amazon Linux/AL2) or `sudo apt install wireguard` (Ubuntu).
3. Generate keypair on server and for each client.
4. Configure `/etc/wireguard/wg0.conf` with `[Interface]` for server and `[Peer]` for clients.
5. Enable IP forwarding and NAT (iptables) if you want clients to reach the rest of VPC.
6. Start service: `sudo systemctl enable --now wg-quick@wg0`
7. Distribute client configs.

WireGuard is simple, performant, and well-suited for developer access.

---

## 6) Allow VPN clients to access EC2 instances — routing & security groups

This section covers what to change in AWS so VPN clients can reach EC2 private IPs.

**A. Security Groups (most common mistake)**

* Edit the **EC2 instance security group** to allow inbound traffic from the VPN client CIDR (example `10.8.0.0/24` or `10.100.0.0/22`), not from the public internet.
* Example for SSH access to private EC2 from VPN: Add inbound rule `Type: SSH, Protocol: TCP, Port: 22, Source: 10.8.0.0/24`.

**B. VPC Route Table (for managed VPN or routed setups)**

* For AWS Client VPN: When you associate subnets, AWS creates ENIs and you add a route in the Client VPN routing table to your VPC CIDR — no manual VPC route change needed.
* For self-managed VPN instance (when not NATing): Add a route in the VPC route table so traffic to the client CIDR (e.g., `10.8.0.0/24`) points to the VPN EC2 instance as the target (instance ID). This tells VPC hosts to send responses back through the VPN instance.

**C. NAT vs Routed**

* **NAT mode:** The VPN server NATs client traffic to its own IP in the VPC. No VPC route changes required. Simpler.
* **Routed mode:** You add a route to VPC for the client CIDR that points at the VPN server. This is transparent and often preferred for larger deployments.

**D. DNS resolution**

* If you want clients to resolve private hostnames, configure DNS: either push the VPC DNS (AmazonProvidedDNS) to clients or run your own DNS resolver.

**E. Firewall on EC2 instances**

* Ensure `ufw`/`firewalld` rules on EC2 allow traffic from VPN CIDR.

---

## 7) Example commands & snippets

(Replace variables with your values)

**Check that a client is connected on server**

```bash
# OpenVPN
sudo systemctl status openvpn@server
sudo tail -f /var/log/syslog | grep openvpn
# WireGuard
sudo wg show
```

**NAT rule (iptables example)**

```bash
VPN_CIDR=10.8.0.0/24
sudo iptables -t nat -A POSTROUTING -s $VPN_CIDR -o eth0 -j MASQUERADE
```

**Route table add (for routed self-managed VPN)**

* Console: VPC > Route Tables > Edit routes > Add route: Destination `10.8.0.0/24` → Target: `i-0123456789abcdef0` (VPN instance)

**Security Group rule for SSH from VPN**

* Inbound: Type = SSH, Source = `10.8.0.0/24`

---

## 8) Troubleshooting

* **Client cannot connect:** Check security group allows VPN port (1194 UDP for OpenVPN) and network ACLs; check server process status; verify EIP is reachable.
* **Client connected but cannot reach EC2:** Check EC2 SG rules, server iptables NAT, VPC route table (for routed mode). Use `tcpdump` on server to see traffic.
* **DNS not resolving private names:** Push VPC DNS to client or configure stub resolver.
* **Multiple clients receive same IP or cannot connect concurrently:** Check client CIDR pool size and server config (max clients).

---

## 9) Security & best practices

* **Prefer managed AWS Client VPN** for production when possible.
* Use **IAM roles** and instance profiles instead of long-lived access keys for automation.
* **Use certificate-based auth or federated auth** (SAML/AD) for user control.
* Rotate certificates and credentials regularly.
* Restrict Security Group access to VPN CIDR only.
* Enable logging and monitoring (CloudWatch logs, VPC Flow Logs).
* Keep the VPN server OS and packages up to date.
* Consider multi-AZ options or HA architecture for production.

---

## 10) Cleanup (if you want to remove everything)

* AWS Client VPN: Disassociate target networks → Delete client VPN endpoint → Remove ACM certificate (if not used elsewhere).
* Self-managed: Terminate EC2 instance → Remove EIP → Remove route entries and security group rules.

---

## Appendix: Quick start commands (OpenVPN on Ubuntu example)

```bash
# example quick setup for testing (Ubuntu)
sudo apt update && sudo apt install -y openvpn easy-rsa iptables-persistent
make-cadir ~/openvpn-ca
cd ~/openvpn-ca
./easyrsa init-pki
./easyrsa build-ca nopass
./easyrsa gen-req server nopass
./easyrsa sign-req server server
./easyrsa gen-dh
openvpn --genkey --secret ta.key
# copy keys to /etc/openvpn and create server.conf (OpenVPN docs)
sudo sysctl -w net.ipv4.ip_forward=1
sudo iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE
sudo systemctl enable --now openvpn@server
```

---

