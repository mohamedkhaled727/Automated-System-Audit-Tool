# How to Create SSL Certificates with OpenSSL
## Requirements 14 & 15 — Step-by-Step Guide

---

## What Are We Doing?

We need to do two things:
1. **Requirement 14**: Create our own **Certificate Authority (CA)** named "Transport Root"
2. **Requirement 15**: Create a **website certificate** for `www.secure-transport.com` that expires in **1 year**, signed by our CA

### What is a Certificate Authority (CA)?
A CA is like a **trusted stamp office**. When a website needs to prove it's legit (HTTPS), it gets a certificate signed by a CA. Browsers trust well-known CAs (like Let's Encrypt, DigiCert). Here, we create our **own private CA** called "Transport Root".

### What is a Website Certificate?
A website certificate is a file that says: "I am www.secure-transport.com and Transport Root CA vouches for me." It contains the domain name, expiry date, and a digital signature from the CA.

---

## Prerequisites

- **XAMPP installed** (it includes OpenSSL at `C:\xampp\apache\bin\openssl.exe`)
- **Open a Command Prompt (cmd)** or **PowerShell**

---

## Step-by-Step Instructions

### Step 0: Set Up Environment

Open **Command Prompt (cmd)** and navigate to your project:

```cmd
cd C:\xampp\htdocs\Assignment_2
mkdir ssl\certs
```

Set the OpenSSL config path (XAMPP needs this):

```cmd
set OPENSSL_CONF=C:\xampp\apache\conf\openssl.cnf
```

> **Note**: If using **PowerShell** instead of cmd, use:
> ```powershell
> $env:OPENSSL_CONF="C:\xampp\apache\conf\openssl.cnf"
> ```

We'll use this shorthand for the OpenSSL command:
```
C:\xampp\apache\bin\openssl.exe
```

---

### Step 1: Generate the CA Private Key

```cmd
C:\xampp\apache\bin\openssl.exe genrsa -out ssl\certs\transport_root_ca.key 4096
```

**What this does:**
- `genrsa` = Generate an RSA private key
- `-out ssl\certs\transport_root_ca.key` = Save it to this file
- `4096` = Key size in bits (4096 is very secure)

**What you get:**
- `transport_root_ca.key` — This is the CA's **private key**. It's like the CA's secret password. Anyone with this key can sign certificates.

---

### Step 2: Create the CA Certificate (Self-Signed)

```cmd
C:\xampp\apache\bin\openssl.exe req -x509 -new -nodes -key ssl\certs\transport_root_ca.key -sha256 -days 3650 -out ssl\certs\transport_root_ca.crt -subj "/C=EG/ST=Cairo/L=Cairo/O=Transport Root/OU=Certificate Authority/CN=Transport Root"
```

**What each part means:**
| Flag | Meaning |
|---|---|
| `req -x509` | Create a self-signed certificate (CA signs itself) |
| `-new` | Create a new certificate request |
| `-nodes` | Don't encrypt the private key with a password |
| `-key ssl\certs\transport_root_ca.key` | Use the private key we just created |
| `-sha256` | Use SHA-256 hashing algorithm (secure) |
| `-days 3650` | Valid for 10 years (3650 days) |
| `-out ssl\certs\transport_root_ca.crt` | Save the certificate to this file |
| `-subj "..."` | The certificate's identity information (see below) |

**The `-subj` field breakdown:**
| Code | Meaning | Value |
|---|---|---|
| `/C=EG` | Country | Egypt |
| `/ST=Cairo` | State | Cairo |
| `/L=Cairo` | City (Locality) | Cairo |
| `/O=Transport Root` | Organization name | **Transport Root** (this is our CA name!) |
| `/OU=Certificate Authority` | Department | Certificate Authority |
| `/CN=Transport Root` | Common Name | **Transport Root** |

**What you get:**
- `transport_root_ca.crt` — This is the CA certificate. It says "I am Transport Root and I can sign other certificates."

> ✅ **Requirement 14 is DONE!** We created a Certificate Authority named "Transport Root".

---

### Step 3: Generate the Server Private Key

```cmd
C:\xampp\apache\bin\openssl.exe genrsa -out ssl\certs\secure_transport.key 2048
```

**What this does:**
- Same as Step 1, but for the **web server** (not the CA)
- We use 2048 bits (standard for server certificates)

**What you get:**
- `secure_transport.key` — The web server's private key

---

### Step 4: Create a Certificate Signing Request (CSR)

```cmd
C:\xampp\apache\bin\openssl.exe req -new -key ssl\certs\secure_transport.key -out ssl\certs\secure_transport.csr -subj "/C=EG/ST=Cairo/L=Cairo/O=Secure Transport/OU=Web Services/CN=www.secure-transport.com"
```

**What this does:**
- Creates a **CSR** — this is like a "please sign my certificate" request
- The CSR says: "I am www.secure-transport.com, please give me a certificate"

**Key part of `-subj`:**
| Code | Value | Why it matters |
|---|---|---|
| `/CN=www.secure-transport.com` | The domain name | This is the domain the certificate is for! |

**What you get:**
- `secure_transport.csr` — The signing request (sent to the CA for signing)

---

### Step 5: Create the Extension File

We need a small config file to add extra info to the certificate:

```cmd
echo subjectAltName=DNS:www.secure-transport.com,DNS:secure-transport.com > ssl\certs\cert_ext.cnf
echo authorityKeyIdentifier=keyid,issuer >> ssl\certs\cert_ext.cnf
echo basicConstraints=CA:FALSE >> ssl\certs\cert_ext.cnf
echo keyUsage=digitalSignature,nonRepudiation,keyEncipherment,dataEncipherment >> ssl\certs\cert_ext.cnf
```

**What each line means:**
| Line | Meaning |
|---|---|
| `subjectAltName=DNS:www.secure-transport.com` | The certificate works for these domain names |
| `authorityKeyIdentifier=keyid,issuer` | Links back to the CA that signed it |
| `basicConstraints=CA:FALSE` | This is NOT a CA certificate (it's a server cert) |
| `keyUsage=...` | What the certificate can be used for (encryption, signing) |

---

### Step 6: Sign the Server Certificate with the CA

```cmd
C:\xampp\apache\bin\openssl.exe x509 -req -in ssl\certs\secure_transport.csr -CA ssl\certs\transport_root_ca.crt -CAkey ssl\certs\transport_root_ca.key -CAcreateserial -out ssl\certs\secure_transport.crt -days 365 -sha256 -extfile ssl\certs\cert_ext.cnf
```

**What each part means:**
| Flag | Meaning |
|---|---|
| `x509 -req` | Process a certificate signing request |
| `-in ssl\certs\secure_transport.csr` | The CSR we want to sign |
| `-CA ssl\certs\transport_root_ca.crt` | The CA certificate (Transport Root) |
| `-CAkey ssl\certs\transport_root_ca.key` | The CA's private key (to create the signature) |
| `-CAcreateserial` | Auto-generate a serial number |
| `-out ssl\certs\secure_transport.crt` | Save the signed certificate here |
| **`-days 365`** | **Valid for 1 year (365 days)** ← This is requirement 15! |
| `-sha256` | Use SHA-256 hashing |
| `-extfile ssl\certs\cert_ext.cnf` | Include the extensions we defined |

**Expected output:**
```
Certificate request self-signature ok
subject=C = EG, ST = Cairo, L = Cairo, O = Secure Transport, OU = Web Services, CN = www.secure-transport.com
```

> ✅ **Requirement 15 is DONE!** We created a certificate for www.secure-transport.com that expires in 1 year.

---

## Step 7: Verify Everything

### Check the CA certificate:
```cmd
C:\xampp\apache\bin\openssl.exe x509 -in ssl\certs\transport_root_ca.crt -text -noout
```

**Look for:**
```
Issuer: C = EG, ST = Cairo, L = Cairo, O = Transport Root, OU = Certificate Authority, CN = Transport Root
Subject: C = EG, ST = Cairo, L = Cairo, O = Transport Root, OU = Certificate Authority, CN = Transport Root
```
> Issuer == Subject means it's **self-signed** (it's a root CA).

### Check the server certificate:
```cmd
C:\xampp\apache\bin\openssl.exe x509 -in ssl\certs\secure_transport.crt -text -noout
```

**Look for:**
```
Issuer: ... O = Transport Root, CN = Transport Root          ← Signed by our CA!
Subject: ... CN = www.secure-transport.com                   ← For this domain!
Not Before: Jun  9 ...                                       ← Start date
Not After : Jun  9 ... (next year)                           ← Expires in 1 year!
Subject Alternative Name: DNS:www.secure-transport.com       ← Domain confirmed
```

### Verify the certificate chain:
```cmd
C:\xampp\apache\bin\openssl.exe verify -CAfile ssl\certs\transport_root_ca.crt ssl\certs\secure_transport.crt
```

**Expected output:**
```
ssl\certs\secure_transport.crt: OK
```

This means: "The server certificate was correctly signed by the Transport Root CA." ✅

---

## Summary of Files Created

| File | What Is It | Created In |
|---|---|---|
| `transport_root_ca.key` | CA private key (secret!) | Step 1 |
| `transport_root_ca.crt` | CA certificate ("I am Transport Root") | Step 2 |
| `secure_transport.key` | Server private key | Step 3 |
| `secure_transport.csr` | Certificate Signing Request ("please sign me") | Step 4 |
| `cert_ext.cnf` | Extension config file | Step 5 |
| `secure_transport.crt` | Signed server certificate (for www.secure-transport.com, 1 year) | Step 6 |

---

## The Big Picture

```
Step 1-2: CREATE THE CA
┌─────────────────────────────────────────────┐
│         "Transport Root" CA                  │
│                                              │
│  transport_root_ca.key  (private key)        │
│  transport_root_ca.crt  (CA certificate)     │
│                                              │
│  "I am Transport Root. Trust me."            │
└──────────────────┬──────────────────────────┘
                   │ Signs
                   ▼
Step 3-6: CREATE THE SERVER CERTIFICATE
┌─────────────────────────────────────────────┐
│    www.secure-transport.com Certificate      │
│                                              │
│  secure_transport.key   (private key)        │
│  secure_transport.csr   (signing request)    │
│  secure_transport.crt   (signed certificate) │
│                                              │
│  "I am www.secure-transport.com.             │
│   Transport Root CA vouches for me.          │
│   Valid for 1 year."                         │
└─────────────────────────────────────────────┘
```

---

## Quick Copy-Paste (All Commands Together)

If you want to redo everything from scratch, here are all the commands in order:

```cmd
cd C:\xampp\htdocs\Assignment_2
set OPENSSL_CONF=C:\xampp\apache\conf\openssl.cnf
mkdir ssl\certs

REM Step 1: CA private key
C:\xampp\apache\bin\openssl.exe genrsa -out ssl\certs\transport_root_ca.key 4096

REM Step 2: CA certificate (Transport Root)
C:\xampp\apache\bin\openssl.exe req -x509 -new -nodes -key ssl\certs\transport_root_ca.key -sha256 -days 3650 -out ssl\certs\transport_root_ca.crt -subj "/C=EG/ST=Cairo/L=Cairo/O=Transport Root/OU=Certificate Authority/CN=Transport Root"

REM Step 3: Server private key
C:\xampp\apache\bin\openssl.exe genrsa -out ssl\certs\secure_transport.key 2048

REM Step 4: CSR for www.secure-transport.com
C:\xampp\apache\bin\openssl.exe req -new -key ssl\certs\secure_transport.key -out ssl\certs\secure_transport.csr -subj "/C=EG/ST=Cairo/L=Cairo/O=Secure Transport/OU=Web Services/CN=www.secure-transport.com"

REM Step 5: Extension file
echo subjectAltName=DNS:www.secure-transport.com,DNS:secure-transport.com > ssl\certs\cert_ext.cnf
echo authorityKeyIdentifier=keyid,issuer >> ssl\certs\cert_ext.cnf
echo basicConstraints=CA:FALSE >> ssl\certs\cert_ext.cnf
echo keyUsage=digitalSignature,nonRepudiation,keyEncipherment,dataEncipherment >> ssl\certs\cert_ext.cnf

REM Step 6: Sign the certificate (valid 1 year)
C:\xampp\apache\bin\openssl.exe x509 -req -in ssl\certs\secure_transport.csr -CA ssl\certs\transport_root_ca.crt -CAkey ssl\certs\transport_root_ca.key -CAcreateserial -out ssl\certs\secure_transport.crt -days 365 -sha256 -extfile ssl\certs\cert_ext.cnf

REM Step 7: Verify
C:\xampp\apache\bin\openssl.exe x509 -in ssl\certs\secure_transport.crt -text -noout
C:\xampp\apache\bin\openssl.exe verify -CAfile ssl\certs\transport_root_ca.crt ssl\certs\secure_transport.crt
```
