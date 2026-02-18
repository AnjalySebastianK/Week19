
# QUIC, TLS 1.2, and TLS 1.3 – Handshake, Performance, and Security

## 1. Overview

This document explains:

- How **QUIC** improves upon **TLS 1.2** and **TLS 1.3**
- How **QUIC integrates TLS 1.3** into its handshake to reduce latency
- A **comparison** of TLS 1.2, TLS 1.3, and QUIC in terms of performance, connection setup, and security
- A **summary table** highlighting QUIC’s improvements over traditional TLS-over-TCP

The focus is on web usage (e.g., HTTPS over TCP vs. HTTPS over QUIC).

---

## 2. Background: TLS over TCP vs. QUIC

### 2.1 TLS 1.2 over TCP

- **Transport:** TCP (reliable, ordered, byte stream).
- **Handshake layering:**  
  1. TCP 3-way handshake (SYN, SYN-ACK, ACK)  
  2. TLS 1.2 handshake (multiple round trips).
- **Typical latency (new connection):**  
  - ~1 RTT for TCP + ~2 RTT for TLS 1.2 ≈ **3 RTT** before application data.
- **Encryption:**  
  - Supports various ciphers; historically allowed weaker/legacy options.
  - Handshake messages partly in cleartext; more complex negotiation.
- **Session resumption:**  
  - Abbreviated handshakes, but still requires TCP handshake and at least 1 RTT for TLS.

### 2.2 TLS 1.3 over TCP

- **Transport:** Still TCP.
- **Handshake improvements:**
  - Simplified handshake: typically **1 RTT** for TLS (vs. 2 RTT in TLS 1.2).
  - Combined with TCP 3-way handshake, effective cost ≈ **2 RTT** for a fresh connection.
- **0-RTT early data (resumption only):**
  - With a previously established session (PSK/session ticket), client can send **early data** in the first flight.
  - Still needs TCP connection establishment first, so “0-RTT” is only for the TLS layer, not the full transport.
- **Security improvements:**
  - Stronger defaults, forward secrecy by design for normal (non-0-RTT) data.
  - Removes many legacy/weak features from TLS 1.2.

### 2.3 QUIC with TLS 1.3

- **Transport:** Runs over **UDP**, but implements its own:
  - Reliability
  - Congestion control
  - Stream multiplexing
  - Connection migration
- **Handshake integration:**
  - QUIC **embeds TLS 1.3** directly into the transport handshake.
  - No separate TCP handshake—transport and crypto are negotiated together.
- **Latency:**
  - Fresh connection: typically **1 RTT** to send encrypted application data.
  - Resumed connection with 0-RTT: **true 0-RTT** for application data (no extra transport handshake).
- **Security:**
  - Uses TLS 1.3 for key exchange and encryption.
  - All QUIC packets (beyond the very first) are encrypted; even metadata is more protected than in TCP+TLS.
- **Multiplexing and head-of-line blocking:**
  - Multiple independent streams within one QUIC connection.
  - Packet loss affects only the streams that depend on that data, not the entire connection (unlike TCP).

---

## 3. How QUIC improves upon TLS 1.2 and TLS 1.3

### 3.1 Latency and connection setup

**TLS 1.2 over TCP:**

- TCP handshake: 1 RTT.
- TLS 1.2 handshake: typically 2 RTT.
- Total: ≈ 3 RTT before first application data on a new connection.

**TLS 1.3 over TCP:**

- TCP handshake: 1 RTT.
- TLS 1.3 handshake: 1 RTT (fresh connection).
- Total: ≈ 2 RTT before first application data (fresh).
- Resumption with 0-RTT:
  - Application data can be sent with ClientHello, but still after TCP handshake.

**QUIC with TLS 1.3:**

- No separate TCP handshake.
- QUIC Initial packet already carries TLS 1.3 ClientHello.
- Fresh connection:
  - Client sends Initial (with ClientHello).
  - Server responds with Initial/Handshake (ServerHello, etc.).
  - After 1 RTT, client can send encrypted application data.
- Resumption with 0-RTT:
  - Client can send **0-RTT application data** in the very first flight.
  - This is **true 0-RTT** for the full connection (transport + crypto).

### 3.2 Performance and head-of-line blocking

**TLS 1.2 and 1.3 over TCP:**

- TCP provides a single ordered byte stream.
- If one packet is lost, subsequent data cannot be delivered to the application until the missing packet is retransmitted and received.
- With HTTP/2 over TCP:
  - Multiple logical streams are multiplexed over one TCP connection.
  - A single lost packet causes **head-of-line blocking** for all streams.

**QUIC:**

- Implements its own stream abstraction on top of UDP.
- Each stream is independently ordered and reliable.
- Loss of a packet affects only the streams that depend on that packet.
- Reduces head-of-line blocking at the transport level, improving performance especially on lossy networks.

### 3.3 Security properties

**TLS 1.2:**

- Supports a wide range of cipher suites, including legacy ones.
- More complex handshake; some parts in cleartext.
- Optional forward secrecy (depends on chosen cipher suite).

**TLS 1.3:**

- Stronger, modern cipher suites only.
- Forward secrecy is mandatory for normal data.
- Handshake is shorter and more robust.
- 0-RTT early data:
  - Not forward-secret.
  - Vulnerable to replay; must be used carefully.

**QUIC with TLS 1.3:**

- Inherits TLS 1.3’s cryptographic strength.
- Encrypts more metadata (e.g., most of the header fields).
- Integrates anti-amplification and address validation mechanisms to mitigate DoS.
- 0-RTT has similar replay/forward secrecy caveats as TLS 1.3, but:
  - QUIC can tie replay protection more closely to transport-level state.

---

## 4. How QUIC integrates TLS 1.3 into its handshake

### 4.1 Layering model

- QUIC treats TLS 1.3 handshake messages as **opaque bytes** carried in special **CRYPTO frames**.
- QUIC defines several packet types:
  - **Initial**: carries the first TLS handshake messages (ClientHello, ServerHello, etc.).
  - **Handshake**: carries later TLS handshake messages.
  - **1-RTT** (short header): carries application data after handshake completion.
- TLS 1.3 runs “inside” QUIC, but QUIC:
  - Provides reliable, ordered delivery for handshake bytes.
  - Uses the keys derived by TLS to protect its own packets.

### 4.2 Handshake flow (fresh connection, simplified)

1. **Client → Server:**
   - Sends QUIC Initial packet with:
     - ClientHello (TLS 1.3) in CRYPTO frames.
   - Some parts are protected with an Initial key derived from connection ID (not yet fully secure, but prevents trivial spoofing).

2. **Server → Client:**
   - Sends Initial + Handshake packets with:
     - ServerHello, EncryptedExtensions, Certificate, CertificateVerify, Finished (TLS 1.3).
   - Derives handshake and application keys.

3. **Client → Server:**
   - Sends Handshake packet with Finished.
   - Switches to 1-RTT keys and can send encrypted application data.

- Result: **1 RTT** to start sending fully encrypted application data.

### 4.3 Handshake flow (resumption with 0-RTT)

1. **Client → Server:**
   - Sends Initial packet with:
     - ClientHello + 0-RTT application data (encrypted with PSK-based early data keys).
2. **Server → Client:**
   - Validates PSK/session ticket.
   - Accepts or rejects 0-RTT data.
   - Completes handshake similarly to fresh connection.

- Result: Application data is sent in the **very first flight**—**true 0-RTT**.

---

## 5. Comparison: TLS 1.2 vs TLS 1.3 vs QUIC

### 5.1 High-level comparison

- **TLS 1.2 over TCP:**
  - Higher latency (more RTTs).
  - Legacy crypto options.
  - No integrated stream multiplexing; head-of-line blocking with HTTP/2.

- **TLS 1.3 over TCP:**
  - Reduced handshake RTT.
  - Stronger, modern cryptography.
  - 0-RTT early data (TLS layer only).
  - Still suffers from TCP head-of-line blocking.

- **QUIC (with TLS 1.3):**
  - Combines transport and crypto handshake.
  - 1-RTT for fresh connections, true 0-RTT for resumption.
  - Stream multiplexing without TCP-level head-of-line blocking.
  - Better protection of metadata and integrated DoS mitigations.

---

## 6. Summary table: QUIC vs traditional TLS-over-TCP

### 6.1 TLS 1.2 vs TLS 1.3 vs QUIC

| Aspect                        | TLS 1.2 over TCP                         | TLS 1.3 over TCP                          | QUIC (with TLS 1.3)                                      |
|------------------------------|------------------------------------------|-------------------------------------------|----------------------------------------------------------|
| Transport                    | TCP                                      | TCP                                       | UDP (custom reliable transport)                          |
| Handshake RTT (fresh)        | ~3 RTT (TCP + TLS)                       | ~2 RTT (TCP + TLS)                        | ~1 RTT (combined QUIC + TLS 1.3)                         |
| 0-RTT support                | No                                       | Yes (TLS layer only, after TCP)           | Yes, **true 0-RTT** (transport + crypto)                 |
| Stream multiplexing          | Via HTTP/2, but TCP-level HoL blocking   | Same as TLS 1.2 (HTTP/2 over TCP)         | Native streams, reduced HoL blocking                     |
| Head-of-line blocking        | Yes, at TCP level                        | Yes, at TCP level                         | Only per-stream; no global TCP HoL blocking              |
| Crypto strength              | Mixed (legacy + modern)                  | Modern, strong, forward-secret by default | Same as TLS 1.3 (modern, strong)                         |
| Metadata protection          | TLS record headers visible; TCP headers  | Similar to TLS 1.2                        | More header fields encrypted/obfuscated                  |
| Connection migration         | No (TCP bound to 4-tuple)                | No                                        | Yes (connection ID allows migration across IP/port)      |
| DoS / amplification controls | Basic TCP mechanisms                     | Similar to TLS 1.2                        | Built-in anti-amplification and address validation       |
| Deployment                   | Very widely deployed                     | Widely deployed                           | Rapidly growing (HTTP/3), requires UDP support           |

### 6.2 Key improvements and benefits of QUIC

| Category          | QUIC Improvement                                      | Practical Benefit                                      |
|-------------------|--------------------------------------------------------|--------------------------------------------------------|
| Latency           | 1-RTT handshake, true 0-RTT resumption                 | Faster page loads, better UX, especially on mobile     |
| Multiplexing      | Independent streams over one connection                | Less impact from packet loss, smoother performance     |
| HoL blocking      | No TCP-level HoL blocking                              | Better throughput on lossy networks                    |
| Security          | TLS 1.3 integrated, more encrypted metadata            | Strong crypto + better privacy                         |
| Mobility          | Connection migration via connection IDs                | Stable connections across network changes (Wi-Fi/LTE)  |
| DoS resistance    | Anti-amplification and address validation              | Reduced risk of reflection/amplification attacks       |

---

## 7. Short conceptual summary

- **TLS 1.2 → TLS 1.3:**  
  - Main gains: simpler, faster, and more secure handshake; modern cryptography; optional 0-RTT early data.

- **TLS 1.3 over TCP → QUIC (with TLS 1.3):**  
  - QUIC goes further by:
    - Eliminating the separate TCP handshake.
    - Integrating TLS 1.3 into the transport handshake.
    - Providing true 0-RTT for the full connection.
    - Reducing head-of-line blocking via native streams.
    - Enhancing security and privacy at the transport layer.

In practice, **HTTP/3 over QUIC** is designed to be the next step after HTTP/2 over TLS/TCP, combining **lower latency**, **better performance under loss**, and **strong security**.

---
