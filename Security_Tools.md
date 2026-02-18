
# SSL/TLS Vulnerability Assessment Using Security Tools  
A Complete Technical Guide

This document explains how to use **Nmap**, **OpenSSL**, and **testssl.sh** to identify SSL/TLS vulnerabilities.  
It includes commands, explanations, findings, and a summary table.

---

# 1. Overview of SSL/TLS Security Tools

## 1.1 Nmap
Nmap is a powerful network scanner with built‑in NSE scripts for SSL/TLS analysis.

Useful scripts:
- `ssl-enum-ciphers` — lists supported ciphers and grades them  
- `ssl-cert` — retrieves certificate details  
- `ssl-dh-params` — checks Diffie‑Hellman strength  
- `ssl-heartbleed` — checks for Heartbleed vulnerability  

---

## 1.2 OpenSSL
OpenSSL provides command‑line tools to:
- Inspect certificates  
- Test SSL/TLS versions  
- Check supported ciphers  
- Validate certificate chains  

---

## 1.3 testssl.sh
A comprehensive SSL/TLS scanner that checks:
- Protocol versions  
- Cipher suites  
- Vulnerabilities (BEAST, CRIME, DROWN, Heartbleed, etc.)  
- Certificate issues  
- Server configuration weaknesses  

---

# 2. Using Nmap for SSL/TLS Assessment

## 2.1 Enumerate Supported Ciphers
```bash
nmap --script ssl-enum-ciphers -p 443 example.com
```

### What it reveals:
- Weak ciphers (RC4, 3DES, EXPORT)  
- Cipher strength grading  
- TLS version support  

---

## 2.2 Retrieve Certificate Information

```bash
nmap --script ssl-cert -p 443 example.com
```

### What it reveals:
- Expiration date  
- Issuer  
- Key size  
- Signature algorithm (e.g., SHA‑1, SHA‑256)  

---

## 2.3 Check for Heartbleed

```bash
nmap --script ssl-heartbleed -p 443 example.com
```

---

## 2.4 Check Diffie‑Hellman Parameters

```bash
nmap --script ssl-dh-params -p 443 example.com
```

### What it reveals:
- Weak DH groups (e.g., 512‑bit, 768‑bit)  
- Vulnerability to Logjam‑style attacks  

---

# 3. Using OpenSSL for SSL/TLS Assessment

## 3.1 View Certificate Details

```bash
openssl s_client -connect example.com:443 -showcerts
```

---

## 3.2 Check Supported TLS Versions

```bash
openssl s_client -connect example.com:443 -tls1_2
openssl s_client -connect example.com:443 -tls1_1
openssl s_client -connect example.com:443 -tls1
openssl s_client -connect example.com:443 -ssl3
```

### Interpretation:
- If the server accepts TLS 1.0 or SSL 3.0 → **insecure**  
- Only TLS 1.2/1.3 should be enabled  

---

## 3.3 Check Supported Ciphers

```bash
openssl ciphers -v 'ALL:eNULL' | column -t
```

---

# 4. Using testssl.sh for Full SSL/TLS Scanning

## 4.1 Run a Full Scan

```bash
./testssl.sh example.com
```

## 4.2 Run a Fast Scan

```bash
./testssl.sh --fast example.com
```

## 4.3 Run Vulnerability Checks Only

```bash
./testssl.sh --vulnerable example.com
```

### What testssl.sh detects:
- BEAST  
- CRIME  
- DROWN  
- Heartbleed  
- POODLE  
- Weak ciphers  
- Protocol downgrade risks  
- Certificate issues  

---

# 5. Identifying SSL/TLS Issues

## 5.1 Weak Ciphers
### Symptoms:
- Presence of RC4, 3DES, EXPORT ciphers  
- TLS 1.0 or SSL 3.0 support  

### Impact:
- Susceptible to brute‑force or statistical attacks  

### Fix:
- Disable weak ciphers in server configuration  
- Enforce AES‑GCM or ChaCha20‑Poly1305  

---

## 5.2 Expired Certificates
### Symptoms:
- Certificate expiration date in the past  

### Impact:
- Browsers reject the connection  
- MITM attacks become easier  

### Fix:
- Renew certificate  
- Automate renewal (e.g., Let’s Encrypt + cron)  

---

## 5.3 Insecure Configurations
Examples:
- Missing intermediate certificates  
- Weak DH parameters  
- SHA‑1 signatures  
- TLS compression enabled  

### Fix:
- Use modern TLS libraries  
- Disable compression  
- Enforce TLS 1.2/1.3  

---

# 6. Explanation of Common Findings

| Issue | Explanation | Impact | Fix |
|-------|-------------|--------|-----|
| **Weak Ciphers** | Server supports outdated algorithms | Decryption, MITM | Disable RC4/3DES, use AES‑GCM |
| **Expired Certificate** | Certificate validity period ended | Browsers reject connection | Renew certificate |
| **SHA‑1 Signature** | Weak hashing algorithm | Certificate forgery | Use SHA‑256+ |
| **TLS 1.0/1.1 Enabled** | Outdated protocol versions | BEAST, downgrade attacks | Enforce TLS 1.2/1.3 |
| **Weak DH Params** | DH < 2048 bits | Logjam attack | Use 2048+ bit DH |
| **Compression Enabled** | Vulnerable to CRIME | Cookie theft | Disable TLS compression |
| **SSLv2/SSLv3 Enabled** | Obsolete protocols | DROWN, POODLE | Disable SSLv2/3 |

---

# 7. Summary

Modern secure configurations should:
- Support only TLS 1.2 and TLS 1.3  
- Disable weak ciphers and protocols  
- Use strong certificates (RSA 2048+, ECDSA)  
- Disable compression  
- Use modern libraries (OpenSSL 1.1.1+, BoringSSL, LibreSSL)  
---
