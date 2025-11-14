# VPN---Demo

Got you!
Here is the **clean, complete, production-ready process** for creating a **Server Certificate for AWS Client VPN** using **EasyRSA**.

You can paste this directly into your **GitHub README**.
It is step-by-step, minimal, correct, and matches AWS requirements exactly.

---

# 🟩 **AWS Client VPN – Server Certificate Creation Guide (EasyRSA)**

This guide explains how to generate an AWS-compatible **Server Certificate** using EasyRSA and import it into **AWS Certificate Manager (ACM)** to use with **AWS Client VPN Endpoint**.

---

# 📌 **Prerequisites**

* Linux system (EC2 or local machine)
* OpenSSL installed
* Git installed

---

# 🟦 **1. Install EasyRSA**

```bash
git clone https://github.com/OpenVPN/easy-rsa.git
cd easy-rsa/easyrsa3
```

---

# 🟦 **2. Initialize Public Key Infrastructure (PKI)**

```bash
./easyrsa init-pki
```

This creates the **pki/** directory that stores keys and certificates.

---

# 🟦 **3. Build Certificate Authority (CA)**

This creates your CA certificate (root certificate).

```bash
./easyrsa build-ca nopass
```

When asked for **Common Name**, enter something like:

```
clientvpn-ca
```

---

# 🟩 **4. Generate the Server Certificate and Private Key (IMPORTANT)**

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

# 🟦 **5. Verify the certificate (Optional but recommended)**

```bash
openssl x509 -text -noout -in pki/issued/server.crt | grep -A2 "Extended Key"
```

Expected output:

```
X509v3 Extended Key Usage:
    TLS Web Server Authentication
```

---

# 🟩 **6. Import the Certificate into AWS Certificate Manager (ACM)**

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

# 🟩 **7. Confirm the certificate in ACM**

You should see:

```
Type: Imported
Status: Issued
In use: No
```

Region must be the same as your Client VPN endpoint (example: **us-east-1**).

---

# 🟦 **8. Use the Certificate in AWS Client VPN Endpoint**

While creating a Client VPN endpoint:

* **Endpoint IP address type:** IPv4
* **Traffic IP address type:** IPv4
* **Authentication options:**
  ✔ Use mutual authentication
* **Server certificate ARN:**
  Choose the certificate you imported
* Continue filling the rest normally

---

# 🟩 **Directory Structure After Generation**

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


