
# SSL/TLS Vulnerabilities — Detailed Technical Overview  
This document explains four major SSL/TLS vulnerabilities:  
**Heartbleed, BEAST, DROWN, and CRIME** — including their causes, exploit mechanisms, impacts, and modern mitigation strategies.

---

# 1. Heartbleed (CVE‑2014‑0160)

## 1.1 Root Cause
Heartbleed was caused by a **bounds‑check failure** in OpenSSL’s implementation of the TLS Heartbeat extension.  
The server trusted the payload length provided by the client without verifying it.

## 1.2 Exploit Scenario
An attacker sends a malicious heartbeat request claiming a large payload size.  
The server responds by returning **up to 64 KB of memory**, which may contain:

- Private keys  
- Usernames/passwords  
- Session cookies  
- Sensitive application data  

## 1.3 Impact
- Full compromise of TLS private keys  
- Ability to decrypt past and future traffic  
- Credential theft  
- Large‑scale data exposure  

## 1.4 Mitigation
- Update OpenSSL to patched versions (1.0.1g or later)  
- Revoke and reissue TLS certificates  
- Regenerate private keys  
- Enable Perfect Forward Secrecy (PFS) to limit long‑term damage  

## 1.5 Real‑World Example
- Major services like **Yahoo**, **GitHub**, and **Cloudflare** were affected.  
- Cloudflare demonstrated that private keys *could* be extracted.

---

# 2. BEAST Attack (Browser Exploit Against SSL/TLS)

## 2.1 Root Cause
BEAST exploited a vulnerability in **TLS 1.0’s CBC mode** due to predictable IVs (initialization vectors).

## 2.2 Exploit Scenario
- Attacker performs a **man‑in‑the‑middle** attack.  
- Injects chosen plaintext into the victim’s encrypted session.  
- Uses JavaScript in the victim’s browser to repeatedly send crafted requests.  
- Recovers secure cookies block by block.

## 2.3 Impact
- Session hijacking  
- Decryption of HTTPS cookies  
- Compromise of authenticated sessions  

## 2.4 Mitigation
- Disable TLS 1.0  
- Use TLS 1.1+ (fixes IV issue)  
- Prefer AEAD ciphers (AES‑GCM, ChaCha20‑Poly1305)  
- Modern browsers implemented client‑side mitigations  

## 2.5 Real‑World Example
- Demonstrated in 2011 by Thai Duong and Juliano Rizzo.  
- Affected nearly all browsers at the time.

---

# 3. DROWN Attack (Decrypting RSA with Obsolete and Weakened eNcryption)

## 3.1 Root Cause
DROWN exploited servers that supported **SSLv2**, even if TLS was used normally.  
SSLv2 had weak export‑grade ciphers and flawed key exchange.

## 3.2 Exploit Scenario
- Attacker sends specially crafted SSLv2 handshake messages.  
- Uses weaknesses in SSLv2 RSA key exchange to decrypt TLS sessions.  
- Works even if SSLv2 is enabled on a *different* service sharing the same certificate/private key.

## 3.3 Impact
- Decryption of TLS sessions  
- Exposure of sensitive data  
- Large‑scale passive decryption possible  

## 3.4 Mitigation
- Disable SSLv2 and SSLv3 entirely  
- Ensure no shared certificates with legacy services  
- Use modern TLS libraries  
- Regenerate keys if compromise is suspected  

## 3.5 Real‑World Example
- Affected ~33% of all HTTPS servers in 2016.  
- Major cloud providers had to patch infrastructure.

---

# 4. CRIME Attack (Compression Ratio Info‑leak Made Easy)

## 4.1 Root Cause
CRIME exploited **TLS‑level compression**.  
By observing compressed ciphertext sizes, attackers inferred secret data (e.g., cookies).

## 4.2 Exploit Scenario
- Attacker injects chosen plaintext into HTTPS requests.  
- Compression reduces size when attacker’s guess matches secret data.  
- By measuring ciphertext length, attacker recovers secrets byte‑by‑byte.

## 4.3 Impact
- Theft of session cookies  
- Account hijacking  
- Exposure of authentication tokens  

## 4.4 Mitigation
- Disable TLS‑level compression  
- Disable SPDY compression (also vulnerable)  
- Use HTTP/2 or HTTP/3 (no compression of sensitive headers)  

## 4.5 Real‑World Example
- Demonstrated in 2012 by the same researchers behind BEAST.  
- Browser vendors quickly disabled TLS compression.

---

# 5. Consolidated Vulnerability Summary Table

| Vulnerability | Root Cause | Exploit Scenario | Impact | Mitigation | Real‑World Case |
|---------------|------------|------------------|---------|------------|------------------|
| **Heartbleed** | Bounds‑check failure in OpenSSL Heartbeat | Malicious heartbeat request leaks memory | Private key theft, data exposure | Patch OpenSSL, regenerate keys, revoke certs | Yahoo, Cloudflare |
| **BEAST** | TLS 1.0 CBC predictable IV | MITM + chosen plaintext via browser | Cookie/session hijacking | Disable TLS 1.0, use AEAD ciphers | 2011 browser attacks |
| **DROWN** | SSLv2 support + shared RSA keys | SSLv2 handshake exploitation | Decrypt TLS sessions | Disable SSLv2/SSLv3, separate certs | 33% of HTTPS servers |
| **CRIME** | TLS compression leaks info via size | Chosen plaintext + size analysis | Cookie theft, account hijacking | Disable TLS compression | Browser vendors patched |

---

# 6. Final Notes
Modern systems mitigate these vulnerabilities by:

- Disabling legacy protocols (SSLv2, SSLv3, TLS 1.0)  
- Using TLS 1.3 exclusively where possible  
- Enforcing AEAD cipher suites  
- Avoiding compression of sensitive data  
- Ensuring libraries (OpenSSL, NSS, BoringSSL) are fully patched  

---
