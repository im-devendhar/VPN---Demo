




# VPN-Demo


<img width="688" height="425" alt="image" src="https://github.com/user-attachments/assets/bfe92fdc-69d7-4ee6-affb-40a12885ee7b" />
 
 ---
 
 🛡️ How AWS Client VPN Checks and Forwards Traffic

When a user connects to the AWS Client VPN, the traffic flow goes through several stages of validation before it reaches the EC2 instance inside the VPC.

🔷 1. Subnet Association Check

The VPN first checks which subnet it is associated with.
The association gives the VPN endpoint a network interface inside the VPC.
Without this, the VPN cannot forward any traffic into the VPC.

🔷 2. Authorization Rule Check

Next, the VPN checks which IP ranges (CIDRs) the user is authorized to access.
If the user's destination IP (e.g., EC2 private IP) is not within the authorized range, the request is rejected.

🔷 3. Route Evaluation

If the destination is authorized, the VPN checks its internal Client VPN route table.
It verifies whether a route exists for the EC2 subnet and maps it to a specific associated subnet.

🔷 4. Security Group Check

After routing is confirmed, traffic reaches the subnet and must pass through Security Groups attached to:

The EC2 instance, and

(If applicable) the Client VPN endpoint network interface

These Security Groups must allow:

Inbound traffic from the Client VPN CIDR (or specific users)

Required ports (SSH: 22, RDP: 3389, HTTP: 80, HTTPS: 443, etc.)

If security groups block the request → the connection fails even if routing is correct.

🔷 5. Traffic Forwarding

Once security groups allow the traffic, it is finally delivered to the EC2 instance.
The EC2 instance responds back through the same path.

---

Absolutely — here is a **clean, professional, README-ready section** describing the exact steps you used to create the **CA**, **Server**, and **Client** certificates on **Amazon Linux** using **EasyRSA 3**.

This is formatted to be copy-paste ready for GitHub.

---

# 📘 **Certificate Creation Guide for AWS Client VPN (EasyRSA on Amazon Linux)**

This guide documents the exact steps used to generate the **Certificate Authority (CA)**, **Server certificate**, and **Client certificate** required for configuring **AWS Client VPN** using **mutual authentication**.

All steps below were performed on an **Amazon Linux EC2 instance**.

---

# 🔧 **1. Install dependencies**

```bash
sudo yum update -y
sudo yum install -y git openssl
```

---

# 📥 **2. Download EasyRSA**

```bash
git clone https://github.com/OpenVPN/easy-rsa.git
cd easy-rsa/easyrsa3
```

---

# 🏗️ **3. Initialize the PKI Environment**

```bash
./easyrsa init-pki
```

This creates the directory:

```
pki/
```

---

# 🏛️ **4. Create the Certificate Authority (CA)**

```bash
./easyrsa build-ca nopass
```

When prompted, enter a CA name (example):

```
Common Name: client-vpn
```

This produces:

* `pki/ca.crt` (CA certificate – used for Client Authentication)
* `pki/private/ca.key` (CA private key – keep safe, never upload to ACM)

---

# 🔐 **5. Generate the Server Certificate and Key**

```bash
./easyrsa build-server-full server nopass
```

Output files:

* `pki/issued/server.crt`
* `pki/private/server.key`
* `pki/ca.crt` (chain)

These 3 files are uploaded to ACM as your **Server Certificate**.

Used as:
➡ **Server certificate ARN** in Client VPN endpoint.

---

# 👤 **6. Generate Client Certificate and Key**

```bash
./easyrsa build-client-full client1 nopass
```

Output files:

* `pki/issued/client1.crt`
* `pki/private/client1.key`
* `pki/ca.crt`

These will be embedded into your final `.ovpn` client config file.

**Do NOT upload client certificates to ACM.**

---

# 📝 **7. Import Certificates into AWS ACM**

## 🔹 **7.1 Import Server Certificate (ACM → Import certificate)**

Upload the following:

| ACM Field         | File                     |
| ----------------- | ------------------------ |
| Certificate body  | `pki/issued/server.crt`  |
| Private key       | `pki/private/server.key` |
| Certificate chain | `pki/ca.crt`             |

This becomes your **Server Certificate ARN**.

---

## 🔹 **7.2 Import CA Certificate (Client Authentication CA)**

This one is uploaded ONLY from the **Client VPN creation page**, not ACM.

Steps:

* Go to **VPC → Client VPN Endpoints → Create**
* Enable **Use mutual authentication**
* In the **Client certificate ARN** section, click **“Upload new client certificate”**
* Paste the contents of:

```
pki/ca.crt
```

No private key is required.

This becomes your **Client Certificate ARN**.

---

# 📦 **8. Final Directory Structure**

```
easy-rsa/
└── easyrsa3/
    └── pki/
        ├── ca.crt
        ├── private/
        │   ├── ca.key
        │   ├── server.key
        │   └── client1.key
        ├── issued/
        │   ├── server.crt
        │   └── client1.crt
        ├── reqs/
        └── ...
```

---





