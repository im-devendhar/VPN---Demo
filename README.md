




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
---

#  **AWS Client VPN – Server Certificate Creation Guide (EasyRSA)**

This guide explains how to generate an AWS-compatible **Server Certificate** using EasyRSA and import it into **AWS Certificate Manager (ACM)** to use with **AWS Client VPN Endpoint**.

---

#  **Prerequisites**

* Linux system (EC2 or local machine)
* OpenSSL installed
* Git installed

---

#  **1. Install EasyRSA**

```bash
git clone https://github.com/OpenVPN/easy-rsa.git
cd easy-rsa/easyrsa3
```

---

#  **2. Initialize Public Key Infrastructure (PKI)**

```bash
./easyrsa init-pki
```

This creates the **pki/** directory that stores keys and certificates.

---

#  **3. Build Certificate Authority (CA)**

This creates your CA certificate (root certificate).

```bash
./easyrsa build-ca nopass
```

When asked for **Common Name**, enter something like:

```
clientvpn-ca
```

---

#  **4. Generate the Server Certificate and Private Key (IMPORTANT)**

This certificate is used by AWS Client VPN Endpoint.

```bash
./easyrsa build-server-full server nopass
```

Output files:

| File                     | Purpose                     |
| ------------------------ | --------------------------- |
| `pki/issued/server.crt`  | Server certificate          |
| `pki/private/server.key` | Private key                 |
| `pki/ca.crt`             | Root CA certificate (chain) |

✔ This certificate contains the correct **Extended Key Usage = TLS Web Server Authentication**, which AWS requires.

---

#  **5. Verify the certificate (Optional but recommended)**

```bash
openssl x509 -text -noout -in pki/issued/server.crt | grep -A2 "Extended Key"
```

Expected output:

```
X509v3 Extended Key Usage:
    TLS Web Server Authentication
```

---

#  **6. Import the Certificate into AWS Certificate Manager (ACM)**

Go to:

**AWS Console → Certificate Manager → Import Certificate**

Upload these:

### Certificate body:

```bash
cat pki/issued/server.crt
```

### Certificate private key:

```bash
cat pki/private/server.key
```

### Certificate chain:

```bash
cat pki/ca.crt
```

Paste the output text into ACM fields.

Then click **Import**.

---

#  **7. Confirm the certificate in ACM**

You should see:

```
Type: Imported
Status: Issued
In use: No
```

Region must be the same as your Client VPN endpoint (example: **us-east-1**).

---

#  **8. Use the Certificate in AWS Client VPN Endpoint**

While creating a Client VPN endpoint:

* **Endpoint IP address type:** IPv4
* **Traffic IP address type:** IPv4
* **Authentication options:**
  ✔ Use mutual authentication
* **Server certificate ARN:**
  Choose the certificate you imported
* Continue filling the rest normally

---

#  **Directory Structure After Generation**

```
easy-rsa/
└── easyrsa3/
    └── pki/
        ├── ca.crt
        ├── issued/
        │   └── server.crt
        └── private/
            └── server.key
```

---



---


