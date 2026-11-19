# Security & Networking — SSL/TLS

> **Phase 4/7 | System Design & Backend | Security**
> **Difficulty:** Intermediate → Advanced
> **Time to complete:** 3–4 hours
> **Status:** ⬜ Not Started

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Why It Exists](#2-why-it-exists)
3. [Fundamentals](#3-fundamentals)
4. [The TLS Handshake](#4-the-tls-handshake)
5. [Visual Architecture](#5-visual-architecture)
6. [Certificates & the Chain of Trust](#6-certificates--the-chain-of-trust)
7. [Cryptography Behind TLS](#7-cryptography-behind-tls)
8. [TLS 1.2 vs TLS 1.3](#8-tls-12-vs-tls-13)
9. [Real-world Usage](#9-real-world-usage)
10. [Performance](#10-performance)
11. [Common Mistakes](#11-common-mistakes)
12. [Best Practices](#12-best-practices)
13. [Debugging](#13-debugging)
14. [Advanced Concepts](#14-advanced-concepts)
15. [Interview Questions](#15-interview-questions)
16. [Hands-on Exercises](#16-hands-on-exercises)
17. [Revision & Cheat Sheet](#17-revision--cheat-sheet)
18. [References](#18-references)

---

## 1. Introduction

### What is it?

**TLS (Transport Layer Security)** is a cryptographic protocol that secures
communication over a network. It provides three guarantees:

- **Confidentiality** — data is encrypted, so eavesdroppers can't read it
- **Integrity** — data can't be tampered with in transit undetected
- **Authentication** — you're talking to the real server, not an impostor

**SSL (Secure Sockets Layer)** is the deprecated predecessor. When people say
"SSL" today, they almost always mean TLS. SSL 2.0 and 3.0 are broken and disabled
everywhere. The term stuck around ("SSL certificate", "SSL/TLS") but the actual
protocol in use is TLS 1.2 or TLS 1.3.

> **Analogy:** TLS is like sending a letter in a tamper-evident locked box, where
> you first verified the recipient's identity with a government-issued ID (the
> certificate), then you both agreed on a secret combination (the handshake) that
> only the two of you know — so nobody in the postal chain can read or alter it.

`HTTPS` = `HTTP` running on top of `TLS`. The "S" is TLS doing its job underneath.

### Why should a senior frontend/full-stack engineer care?

| Scenario | Why TLS knowledge matters |
|---|---|
| Debugging `ERR_CERT_*` errors | You need to read cert chains and expiry |
| System design interviews | "How do you secure the connection?" is a standard follow-up |
| Setting up CDN / load balancer | TLS termination, SNI, cert management |
| API security | mTLS between microservices, certificate pinning |
| Performance tuning | Handshake latency, session resumption, 0-RTT |
| Mixed-content bugs | HTTPS pages blocking HTTP resources |

Interviewers at Atlassian, Adobe, Salesforce expect you to explain the handshake,
the chain of trust, and the difference between symmetric and asymmetric crypto in
this context — not at a cryptographer's depth, but at an architect's depth.

---

## 2. Why It Exists

### The problem TLS had to solve

The internet was built on plaintext. HTTP, SMTP, FTP — all send data as readable
text across many intermediate hops (your router, ISP, backbone routers, the
destination's network). Any of these can:

- **Read** your data (passwords, cookies, messages) → no confidentiality
- **Modify** your data (inject ads, malware) → no integrity
- **Impersonate** the server (phishing, MITM) → no authentication

```
Without TLS (plaintext HTTP):

You → [Router] → [ISP] → [Backbone] → [Server's Network] → Server
        👁         👁          👁              👁
     everyone in the path can read and modify everything
```

### Three threats, three solutions

| Threat | TLS solution | Crypto primitive |
|---|---|---|
| Eavesdropping | Encryption | Symmetric ciphers (AES) |
| Tampering | Message authentication | MAC / AEAD (GCM) |
| Impersonation | Certificates + signatures | Asymmetric crypto (RSA/ECDSA) + PKI |

### Engineering Perspective

> **Why was this designed this way?**
> TLS combines asymmetric and symmetric cryptography deliberately. Asymmetric
> crypto (public/private keys) solves the "how do two strangers agree on a secret
> over an open channel" problem — but it's slow. Symmetric crypto (one shared key)
> is fast but needs a pre-shared secret. TLS uses asymmetric crypto *only* during
> the handshake to securely establish a symmetric key, then uses fast symmetric
> crypto for the actual data. Best of both worlds.

> **What are the alternatives?**
> - **IPsec / VPN** — secures at the network layer, transparent to apps, but
>   heavier to deploy and manage.
> - **Application-layer encryption** (e.g. encrypting the payload yourself) —
>   doesn't authenticate the endpoint or protect metadata.
> - **SSH** — great for terminal/tunnels, not for web browsing at scale.
> TLS won for the web because it's app-agnostic, backed by a global PKI, and
> baked into every browser and server.

> **What trade-offs does it introduce?**
> - **Latency** — the handshake adds round trips before data flows (1-RTT in
>   TLS 1.3, 2-RTT in TLS 1.2).
> - **CPU cost** — encryption/decryption uses CPU (mostly negligible now with
>   hardware AES).
> - **Operational complexity** — certificates expire, must be renewed, rotated,
>   and their private keys protected. Expired certs are a top cause of outages.

> **How would a Staff Engineer think about this?**
> "Where do I terminate TLS — at the CDN edge, the load balancer, or the service?
> Do I need end-to-end encryption (mTLS between services) or is edge termination
> enough? How do I automate cert rotation so a human forgetting to renew never
> takes down production? What's my minimum TLS version and cipher policy?"

---

## 3. Fundamentals

### Core Concepts & Terminology

#### Symmetric encryption
One shared key encrypts and decrypts. Fast. Example: **AES-256-GCM**. Problem: how
do both parties get the same key without an attacker seeing it? That's what the
handshake solves.

#### Asymmetric encryption (public-key cryptography)
A key **pair**: a public key (shared with everyone) and a private key (kept secret).
Data encrypted with the public key can only be decrypted with the private key, and
a signature made with the private key can be verified with the public key.
Examples: **RSA**, **ECDSA**. Slow — used only in the handshake.

#### Key exchange
The mechanism by which both sides derive the same symmetric session key without
ever sending it over the wire. Modern TLS uses **(EC)DHE** — Ephemeral
(Elliptic-Curve) Diffie-Hellman.

#### Certificate (X.509)
A digital document binding a public key to an identity (a domain name). Signed by
a Certificate Authority. Contains: subject (domain), public key, issuer, validity
dates, and the CA's signature.

#### Certificate Authority (CA)
A trusted organization (DigiCert, Let's Encrypt, GlobalSign) that verifies domain
ownership and issues signed certificates. Browsers and OSes ship with a **trust
store** of root CA certificates.

#### Chain of trust
Your server's cert is signed by an intermediate CA, whose cert is signed by a root
CA, whose cert is in the browser's trust store. The browser walks this chain to
verify trust.

#### Cipher suite
A negotiated bundle of algorithms used for one connection. Example:
`TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256` = ECDHE key exchange, RSA authentication,
AES-128-GCM encryption, SHA-256 for the handshake.

#### AEAD (Authenticated Encryption with Associated Data)
Modern cipher mode (like AES-GCM, ChaCha20-Poly1305) that encrypts AND
authenticates in one operation — provides both confidentiality and integrity.

#### PKI (Public Key Infrastructure)
The whole ecosystem of CAs, certificates, trust stores, and revocation mechanisms
that makes web authentication work at global scale.

#### Session resumption
Reusing a previously negotiated secret to skip the full handshake on reconnection.
Reduces latency. Done via session IDs, session tickets, or TLS 1.3 PSK.

#### Forward secrecy (PFS)
A property where compromising the server's long-term private key does NOT let an
attacker decrypt past recorded sessions — because each session used an ephemeral
key that was thrown away. Achieved with ephemeral Diffie-Hellman (DHE/ECDHE).

### Where TLS sits in the stack

```
┌─────────────────────────────────┐
│  Application  (HTTP, SMTP, gRPC) │  ← your data
├─────────────────────────────────┤
│  TLS          (encryption)       │  ← this lesson
├─────────────────────────────────┤
│  Transport    (TCP)              │  ← reliable delivery
├─────────────────────────────────┤
│  Network      (IP)               │  ← routing
└─────────────────────────────────┘
```

TLS sits between the application and transport layers. HTTP hands its bytes to
TLS, TLS encrypts them and hands them to TCP.

---

## 4. The TLS Handshake

The handshake is where the two parties authenticate each other and agree on a
shared symmetric key. This is the heart of TLS.

### TLS 1.2 handshake (2 round trips)

```
Client                                                     Server
  │                                                          │
  │ ── ClientHello ────────────────────────────────────────►│
  │    (TLS versions, cipher suites, client random, SNI)     │
  │                                                          │
  │◄──────────────────────── ServerHello ──────────────────│
  │    (chosen version, chosen cipher, server random)        │
  │◄──────────────────────── Certificate ──────────────────│
  │    (server's cert chain)                                 │
  │◄──────────────────────── ServerKeyExchange ────────────│
  │    (ECDHE params, signed with private key)               │
  │◄──────────────────────── ServerHelloDone ──────────────│
  │                                                          │
  │  [client verifies cert, checks CA chain, expiry, domain] │
  │                                                          │
  │ ── ClientKeyExchange ──────────────────────────────────►│
  │    (client's ECDHE public param)                         │
  │ ── ChangeCipherSpec ───────────────────────────────────►│
  │ ── Finished (encrypted) ───────────────────────────────►│
  │                                                          │
  │◄──────────────────────── ChangeCipherSpec ─────────────│
  │◄──────────────────────── Finished (encrypted) ─────────│
  │                                                          │
  │ ═══════════ Encrypted application data (AES) ═══════════ │
```

**Step by step:**

1. **ClientHello** — client says: "Here are the TLS versions and cipher suites I
   support, here's a random number, and (via SNI) the hostname I want."
2. **ServerHello** — server picks the strongest mutually-supported version and
   cipher, sends its own random number.
3. **Certificate** — server sends its cert chain to prove its identity.
4. **ServerKeyExchange** — server sends its ephemeral Diffie-Hellman public
   parameter, signed with its private key (proving it owns the cert).
5. **Client verifies the certificate** — checks the CA signature chain, that the
   cert isn't expired, and that the domain matches (SAN).
6. **ClientKeyExchange** — client sends its own DH public parameter.
7. Both sides now independently compute the **same shared secret** (the "pre-master
   secret") and derive session keys from it + the two randoms.
8. **ChangeCipherSpec / Finished** — both switch to encryption and send a Finished
   message (a MAC of the whole handshake) so each can verify the handshake wasn't
   tampered with.
9. **Application data** flows, encrypted with the fast symmetric key.

### TLS 1.3 handshake (1 round trip)

TLS 1.3 slashed the round trips by having the client guess the key exchange
parameters up front.

```
Client                                                     Server
  │                                                          │
  │ ── ClientHello ────────────────────────────────────────►│
  │    (versions, ciphers, client random,                    │
  │     + KeyShare: client's ECDHE public key already!)      │
  │                                                          │
  │◄──────────────────────── ServerHello ──────────────────│
  │    (chosen cipher, server random, server's KeyShare)     │
  │◄──────────────────────── {EncryptedExtensions} ────────│
  │◄──────────────────────── {Certificate} ────────────────│
  │◄──────────────────────── {CertificateVerify} ──────────│
  │◄──────────────────────── {Finished} ───────────────────│
  │    ({ } = already encrypted!)                            │
  │                                                          │
  │ ── {Finished} ─────────────────────────────────────────►│
  │                                                          │
  │ ═══════════ Encrypted application data ═════════════════ │
```

Because the client sends its `KeyShare` in the very first message, the server can
compute the shared secret immediately and start encrypting on its first response.
Only **1 round trip** before data flows. TLS 1.3 also removed all the legacy weak
options (static RSA key exchange, RC4, MD5, SHA-1, CBC modes), so every 1.3
connection has forward secrecy by design.

### 0-RTT (zero round trip) resumption — TLS 1.3

On a *reconnection*, the client can send application data in the very first packet
using a pre-shared key from the previous session. Zero handshake latency. The
catch: 0-RTT data is vulnerable to **replay attacks**, so it must only be used for
idempotent requests (GET, not POST that charges a credit card).

---

## 5. Visual Architecture

### The two-phase model: asymmetric then symmetric

```
┌──────────────────────────────────────────────────────────────┐
│  PHASE 1 — HANDSHAKE  (slow, asymmetric crypto)               │
│                                                                │
│   Client ◄──── certificate + signatures + key exchange ────► Server │
│                                                                │
│   Purpose: authenticate the server + agree on a shared        │
│            symmetric session key WITHOUT sending it in clear   │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│  PHASE 2 — DATA TRANSFER  (fast, symmetric crypto)            │
│                                                                │
│   Client ◄════ AES-GCM encrypted with session key ════► Server │
│                                                                │
│   Purpose: encrypt the actual HTTP traffic quickly            │
└──────────────────────────────────────────────────────────────┘
```

### TLS termination in a real deployment

```
                    ┌─────────────── TLS terminated here ──────────┐
                    │                                              │
Browser ══TLS══► CDN Edge ══TLS══► Load Balancer ──plaintext──► App Server
(HTTPS)          (Cloudflare)      (ALB / Nginx)   (internal net)  (Node/Spring)

  Option A: terminate at CDN edge      → lowest latency, CDN sees plaintext
  Option B: terminate at load balancer → common; internal network trusted
  Option C: end-to-end / mTLS          → re-encrypt to backend; zero-trust
```

**TLS termination** = the point where the encrypted connection is decrypted. After
termination, traffic may travel in plaintext over a trusted internal network, or be
re-encrypted for a zero-trust architecture.

### The chain of trust

```
┌────────────────────┐
│  Root CA cert      │  ← in browser/OS trust store (self-signed, pre-installed)
│  (DigiCert Root)   │
└─────────┬──────────┘
          │ signs
          ▼
┌────────────────────┐
│ Intermediate CA    │  ← sent by server, bridges root and leaf
│ (DigiCert TLS RSA) │
└─────────┬──────────┘
          │ signs
          ▼
┌────────────────────┐
│ Leaf / server cert │  ← your actual certificate for example.com
│ (CN=example.com)   │
└────────────────────┘

Browser verifies bottom-up: leaf → intermediate → root (found in trust store ✅)
```

---

## 6. Certificates & the Chain of Trust

### What's inside an X.509 certificate

```
Certificate:
    Version: 3
    Serial Number: 0a:1b:2c:...
    Signature Algorithm: sha256WithRSAEncryption
    Issuer:  C=US, O=Let's Encrypt, CN=R3          ← who signed it
    Validity:
        Not Before: Sep 1 2026
        Not After : Nov 30 2026                     ← expiry (renew before this!)
    Subject: CN=example.com                         ← who it's for
    Subject Public Key Info:
        Public Key: (2048 bit RSA / or EC key)      ← the server's public key
    X509v3 Subject Alternative Name (SAN):          ← domains it covers
        DNS:example.com, DNS:www.example.com
    Signature: <CA's signature over all the above>  ← the CA vouching for it
```

The **SAN** field is what browsers actually check for the domain match (the old
`Common Name`-based matching is deprecated). A cert can cover multiple domains via
multiple SAN entries, or all subdomains via a **wildcard** (`*.example.com`).

### How verification works

When your browser receives the server's cert chain, it checks:

1. **Signature chain** — each cert is validly signed by the next one up, ending at
   a root CA in the trust store.
2. **Not expired** — current time is within the validity window.
3. **Domain match** — the SAN includes the hostname you connected to.
4. **Not revoked** — via CRL (Certificate Revocation List) or OCSP (Online
   Certificate Status Protocol / OCSP stapling).
5. **Proof of private key** — the server signed the handshake with the private key
   matching the cert's public key (this happens in ServerKeyExchange /
   CertificateVerify).

If any check fails → `NET::ERR_CERT_*` and the browser blocks the connection.

### Certificate validation levels

| Level | What's verified | Use case |
|---|---|---|
| **DV** (Domain Validated) | Control of the domain | Most sites; Let's Encrypt |
| **OV** (Organization Validated) | Domain + org identity | Business sites |
| **EV** (Extended Validation) | Rigorous legal vetting | Banks (no longer shows green bar) |

### Certificate revocation

If a private key is compromised, the cert must be invalidated before it expires:

- **CRL** — a big list of revoked serial numbers. Bulky, slow.
- **OCSP** — client asks the CA "is this cert still valid?" in real time. Adds
  latency and a privacy leak (CA learns what you browse).
- **OCSP stapling** — the *server* fetches a signed OCSP response periodically and
  "staples" it to the handshake, so the client doesn't have to ask the CA.
  Best-practice approach.

### mTLS (mutual TLS)

Normal TLS authenticates only the *server*. In **mTLS**, the *client* also presents
a certificate, so both sides authenticate each other. Common for:

- Service-to-service auth in microservices / service mesh (Istio, Linkerd)
- Zero-trust internal networks
- High-security API access (banking, B2B)

```
Client ── presents client cert ──► Server verifies client cert
Client ◄── presents server cert ── Server
   both authenticated → connection established
```

---

## 7. Cryptography Behind TLS

You don't need to be a cryptographer, but you should be able to explain these
building blocks and *why* TLS uses each.

### Symmetric vs asymmetric — the core trade-off

| | Symmetric | Asymmetric |
|---|---|---|
| Keys | One shared secret | Public + private pair |
| Speed | Very fast | ~100–1000x slower |
| Example | AES-256-GCM, ChaCha20 | RSA, ECDSA, ECDHE |
| Problem it has | Key distribution | Performance |
| Role in TLS | Encrypts the actual data | Handshake: auth + key exchange |

TLS uses asymmetric crypto to *bootstrap* a symmetric key, then does the heavy
lifting with symmetric crypto. This is called a **hybrid cryptosystem**.

### Diffie-Hellman key exchange (the "magic")

DH lets two parties derive a shared secret over a public channel without ever
transmitting it. Simplified:

```
Public (everyone sees): a base g and modulus p

Client picks secret a  →  sends  g^a mod p  ─────►
                       ◄─────  sends  g^b mod p     Server picks secret b

Client computes: (g^b)^a mod p  =  g^(ab) mod p   ┐
                                                   ├── SAME shared secret!
Server computes: (g^a)^b mod p  =  g^(ab) mod p   ┘

An eavesdropper sees g^a and g^b but CANNOT feasibly compute g^(ab).
```

**ECDHE** is the elliptic-curve version — same idea, smaller keys, faster. The "E"
= **ephemeral**: a fresh key pair per connection, then discarded. That's what gives
**forward secrecy** — recording today's traffic and stealing the server key next
year still won't decrypt it, because the ephemeral keys are gone.

### Digital signatures (authentication)

The server proves it owns the certificate by signing handshake data with its
**private key**. The client verifies with the **public key** in the cert. Since
only the real server has the private key, this defeats impersonation.

```
Server: signature = sign(handshake_data, private_key)
Client: verify(handshake_data, signature, public_key) → true/false
```

### Hashing & MAC (integrity)

A hash (SHA-256) produces a fixed fingerprint of data — any change flips the hash.
A **MAC** (or AEAD tag) combines a hash with the secret key so an attacker can't
just recompute it after tampering. Modern TLS uses **AEAD** ciphers (AES-GCM,
ChaCha20-Poly1305) that encrypt and authenticate in one step.

### Reading a cipher suite

```
TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
 │    │    │        │    │   │    │
 │    │    │        │    │   │    └── SHA-256 for the handshake PRF/HKDF
 │    │    │        │    │   └─────── GCM = AEAD mode (encrypt + authenticate)
 │    │    │        │    └─────────── 128-bit key
 │    │    │        └──────────────── AES cipher for data
 │    │    └───────────────────────── RSA = server authentication method
 │    └────────────────────────────── ECDHE = key exchange (forward secrecy)
 └─────────────────────────────────── protocol
```

TLS 1.3 simplified this — suites only name the symmetric cipher + hash (e.g.
`TLS_AES_128_GCM_SHA256`), because key exchange and signatures are negotiated
separately and are always forward-secret.

---

## 8. TLS 1.2 vs TLS 1.3

| Aspect | TLS 1.2 (2008) | TLS 1.3 (2018) |
|---|---|---|
| Handshake round trips | 2-RTT | 1-RTT (0-RTT on resume) |
| Forward secrecy | Optional (if ECDHE chosen) | Mandatory |
| Cipher suites | ~37, many weak ones | 5 strong AEAD suites only |
| Static RSA key exchange | Allowed (no PFS) | Removed |
| Weak crypto (RC4, MD5, SHA-1, CBC) | Allowed | Removed |
| Handshake encryption | Mostly in clear | Certificate onward is encrypted |
| Renegotiation | Yes (attack surface) | Removed |

**Why TLS 1.3 matters in interviews:** it's faster (fewer round trips = less
latency, big deal for mobile), and more secure by *removing* footguns rather than
adding options. If asked "how would you improve TLS handshake latency," the answers
are: use TLS 1.3, enable session resumption / 0-RTT, and use OCSP stapling.

Both TLS 1.0 and 1.1 are **deprecated** (removed from browsers in 2020). Minimum
acceptable version today is **TLS 1.2**; prefer **1.3**.

---

## 9. Real-world Usage

### HTTPS in the browser

When you type `https://example.com`:

1. DNS resolves the domain to an IP.
2. TCP connection opens (SYN, SYN-ACK, ACK).
3. **TLS handshake** runs (this lesson).
4. Encrypted HTTP request/response flows.
5. Browser shows the padlock 🔒 if the cert validates.

The padlock means the connection is *encrypted and the server is authenticated* —
it does NOT mean the site is trustworthy (phishing sites can have valid certs too).

### Let's Encrypt + automated renewal

Free, automated DV certs. The `certbot` tool (or built-in ACME clients in Caddy,
Traefik, nginx) automatically:
- Proves domain control (HTTP-01 or DNS-01 challenge)
- Fetches a 90-day cert
- Auto-renews before expiry

This solved the #1 cause of TLS outages: humans forgetting to renew.

### In your stack (Angular / Node)

```javascript
// Node.js HTTPS server
const https = require('https');
const fs = require('fs');

const server = https.createServer({
  key:  fs.readFileSync('privkey.pem'),   // private key — NEVER commit this
  cert: fs.readFileSync('fullchain.pem'),  // leaf + intermediate certs
  minVersion: 'TLSv1.2',                   // reject old protocols
}, app);

server.listen(443);
```

```typescript
// Angular — enforce HTTPS + avoid mixed content
// All API/asset URLs must be https:// on an https:// page,
// or the browser blocks them as "mixed content".
// Set a Content-Security-Policy to upgrade insecure requests:
//   Content-Security-Policy: upgrade-insecure-requests
```

In practice, most Angular/Node apps sit behind a CDN or load balancer that handles
TLS termination — you rarely configure the handshake yourself, but you must
understand it to debug and design.

### HSTS (HTTP Strict Transport Security)

A response header that tells the browser "always use HTTPS for this domain, never
try HTTP again":

```
Strict-Transport-Security: max-age=63072000; includeSubDomains; preload
```

Prevents SSL-stripping attacks where a MITM downgrades the first HTTP request.

---

## 10. Performance

### The cost of the handshake

```
TLS 1.2 full handshake:   2 round trips before data  (~2 × RTT latency)
TLS 1.3 full handshake:   1 round trip                (~1 × RTT latency)
TLS 1.3 resumption (0-RTT): 0 round trips              (data in first packet)
```

On a connection with 100ms RTT, going from TLS 1.2 to 1.3 saves ~100ms per new
connection — meaningful on mobile and for first-visit page loads.

### Session resumption

Avoids the full handshake on reconnection by reusing an earlier secret:

| Mechanism | How | TLS version |
|---|---|---|
| Session IDs | Server caches the session, client sends the ID | 1.2 |
| Session tickets | Server encrypts the session state, client stores it | 1.2 |
| PSK (pre-shared key) | Unified resumption mechanism | 1.3 |

### Other optimizations

- **OCSP stapling** — server bundles the revocation status, saving the client a
  separate round trip to the CA.
- **ECDSA certs over RSA** — smaller keys, faster signatures, less CPU.
- **Hardware AES (AES-NI)** — CPU instructions make symmetric encryption nearly
  free. TLS overhead on data transfer is negligible on modern hardware.
- **Connection reuse / HTTP/2 / HTTP/3** — amortize one handshake over many
  requests. HTTP/3 (QUIC) integrates TLS 1.3 into the transport for even faster
  connection setup.

> "TLS is expensive" is a myth from the 2000s. With TLS 1.3, session resumption,
> AES-NI, and connection reuse, the overhead is a few milliseconds. Google enabled
> HTTPS everywhere with negligible CPU cost.

---

## 11. Common Mistakes

### Mistake 1 — Letting certificates expire

The single most common TLS outage. A cert expires, every client gets
`ERR_CERT_DATE_INVALID`, the site is down.
**Fix:** automate renewal (Let's Encrypt + certbot / ACME), and monitor expiry with
alerts weeks in advance.

### Mistake 2 — Forgetting the intermediate certificate

The server sends only its leaf cert, not the intermediate. Some clients (that don't
cache the intermediate) fail to build the chain.
**Fix:** always serve the **full chain** (`fullchain.pem` = leaf + intermediates),
not just the leaf.

### Mistake 3 — Mixed content

An HTTPS page loads a script/image over HTTP. Browsers block it (active mixed
content) or warn (passive), breaking the page.
**Fix:** use protocol-relative or `https://` URLs everywhere, add
`Content-Security-Policy: upgrade-insecure-requests`.

### Mistake 4 — Committing private keys to git

The private key (`privkey.pem`, `.key`) is the crown jewel. If it leaks, anyone can
impersonate your server. Committing it to a repo is a critical incident.
**Fix:** keep keys in a secrets manager (Vault, AWS Secrets Manager, KMS), add
`*.key`/`*.pem` to `.gitignore`, rotate immediately if exposed.

### Mistake 5 — Supporting weak protocols/ciphers

Leaving TLS 1.0/1.1, RC4, or export ciphers enabled exposes you to POODLE, BEAST,
and downgrade attacks.
**Fix:** set `minVersion: TLSv1.2`, prefer TLS 1.3, use a modern cipher list (e.g.
Mozilla's "Intermediate" or "Modern" config).

### Mistake 6 — Disabling cert validation "to make it work"

```javascript
// NEVER do this in production
process.env.NODE_TLS_REJECT_UNAUTHORIZED = '0';   // ❌ disables ALL cert checks
// or in fetch/axios: rejectUnauthorized: false     // ❌ opens you to MITM
```
This turns off the authentication guarantee entirely. If you hit a cert error, fix
the cert or trust the correct CA — don't disable validation.

---

## 12. Best Practices

1. **Use TLS 1.3, minimum TLS 1.2.** Disable 1.0/1.1 and SSLv3 entirely.
2. **Automate certificate renewal.** Let's Encrypt + ACME. Never renew by hand.
3. **Monitor cert expiry** with alerts at 30/14/7 days out.
4. **Serve the full chain** (leaf + intermediates), not just the leaf cert.
5. **Enable HSTS** with a long `max-age` and consider preloading.
6. **Enable OCSP stapling** to speed up validation and protect privacy.
7. **Use forward-secret cipher suites** (ECDHE) — mandatory in 1.3 anyway.
8. **Protect private keys** — secrets manager / HSM, never in source control,
   restrictive file permissions (`chmod 600`).
9. **Prefer ECDSA certificates** for performance where clients support them.
10. **Redirect all HTTP → HTTPS** and use `upgrade-insecure-requests`.
11. **Consider mTLS for internal service-to-service** traffic (zero trust).
12. **Test your config** with SSL Labs / testssl.sh and aim for an A/A+ grade.

---

## 13. Debugging

### Inspect a server's certificate from the CLI

```bash
# View the full chain and handshake details
openssl s_client -connect example.com:443 -servername example.com

# Just the cert's key fields
openssl s_client -connect example.com:443 -servername example.com </dev/null \
  | openssl x509 -noout -subject -issuer -dates -ext subjectAltName

# Check a local cert file's expiry
openssl x509 -in fullchain.pem -noout -dates

# Verify a chain builds correctly
openssl verify -CAfile chain.pem leaf.pem
```

### Common browser errors and what they mean

| Error | Meaning | Fix |
|---|---|---|
| `ERR_CERT_DATE_INVALID` | Cert expired or clock wrong | Renew cert / fix system clock |
| `ERR_CERT_AUTHORITY_INVALID` | CA not trusted / missing intermediate | Serve full chain; trust CA |
| `ERR_CERT_COMMON_NAME_INVALID` | Domain not in SAN | Reissue cert with correct SAN |
| `ERR_SSL_VERSION_OR_CIPHER_MISMATCH` | No common protocol/cipher | Enable TLS 1.2+, modern ciphers |
| `ERR_SSL_PROTOCOL_ERROR` | Handshake failed | Check server config / logs |
| Mixed content warning | HTTP resource on HTTPS page | Switch resource to HTTPS |

### DevTools

- **Security tab** (Chrome) → shows the cert, chain, protocol, and cipher for the
  page and each resource.
- **Network tab** → click a request → Security section shows the connection details.

### Online tools

- **SSL Labs** (ssllabs.com/ssltest) — grades your config A+ to F, lists issues.
- **testssl.sh** — CLI scanner for protocols, ciphers, vulnerabilities.
- **crt.sh** — search Certificate Transparency logs for all certs issued for a
  domain (useful to spot rogue certs).

---

## 14. Advanced Concepts

### SNI (Server Name Indication)

One IP can host many HTTPS sites. But the server needs to know *which* cert to send
before the encrypted connection exists. **SNI** puts the target hostname in the
ClientHello (in plaintext) so the server picks the right cert. **ESNI/ECH**
(Encrypted Client Hello) encrypts even this, closing a privacy leak.

### Certificate pinning

The client hardcodes which cert/public key or CA to trust for a domain, rejecting
even otherwise-valid certs. Defeats MITM via rogue CAs. Common in mobile apps.
Risky on the web (a bad pin bricks your site) — HPKP was deprecated; use
`Expect-CT` / Certificate Transparency instead.

### Certificate Transparency (CT)

Public, append-only logs of every cert issued. Browsers require certs to be logged.
Lets domain owners detect mis-issued or rogue certs for their domain (via crt.sh or
monitoring services).

### TLS termination vs passthrough vs re-encryption

- **Termination** — LB/CDN decrypts; backend gets plaintext (trusted network).
- **Passthrough** — LB forwards encrypted bytes; backend terminates (LB can't
  inspect/route on content).
- **Re-encryption / end-to-end** — LB terminates then opens a *new* TLS connection
  to the backend (zero-trust; content-based routing + encryption in transit).

### QUIC / HTTP/3

HTTP/3 runs over **QUIC** (on UDP), which builds TLS 1.3 directly into the transport
handshake — combining transport and crypto setup, eliminating head-of-line blocking,
and enabling faster connection establishment and migration across networks.

### Post-quantum TLS

Quantum computers could eventually break RSA and ECDH. The industry is rolling out
**hybrid key exchange** (e.g. `X25519MLKEM768`) combining classical ECDHE with a
post-quantum KEM, so a future quantum attacker can't decrypt today's recorded
traffic ("harvest now, decrypt later"). Already shipping in Chrome and major CDNs.

---

## 15. Interview Questions

### Beginner

**Q: What is the difference between SSL and TLS?**
> TLS is the modern, secure successor to SSL. SSL 2.0/3.0 are deprecated and
> broken. "SSL" persists as a colloquial term (SSL certificate) but every secure
> connection today actually uses TLS 1.2 or 1.3.

**Q: What three guarantees does TLS provide?**
> Confidentiality (encryption), integrity (tamper detection), and authentication
> (you're talking to the real server, verified via its certificate).

**Q: What does the padlock icon actually mean?**
> The connection is encrypted and the server's identity is authenticated by a
> trusted CA. It does NOT mean the website itself is trustworthy — phishing sites
> can legitimately obtain valid certificates.

### Intermediate

**Q: Walk me through the TLS handshake.**
> ClientHello (versions, ciphers, random, SNI) → ServerHello (chosen cipher,
> random) → server sends its certificate and ephemeral key-exchange params signed
> with its private key → client verifies the cert chain and domain → both do ECDHE
> to derive the same shared secret → both switch to symmetric encryption and send
> Finished messages to confirm integrity → encrypted application data flows.

**Q: Why does TLS use both asymmetric and symmetric encryption?**
> Asymmetric crypto solves key exchange between strangers over an open channel but
> is slow. Symmetric crypto is fast but needs a pre-shared key. TLS uses asymmetric
> crypto in the handshake to securely establish a symmetric session key, then uses
> fast symmetric crypto (AES-GCM) for the bulk data. A hybrid cryptosystem.

**Q: What is forward secrecy and how is it achieved?**
> A property where compromising the server's long-term private key can't decrypt
> past sessions. Achieved with ephemeral Diffie-Hellman (ECDHE): each session uses
> a fresh key pair that's discarded afterward, so there's nothing stored to decrypt
> recorded traffic later. Mandatory in TLS 1.3.

**Q: How does certificate validation work?**
> The browser walks the chain: the leaf cert is signed by an intermediate CA,
> signed by a root CA that's in the browser's trust store. It checks the signature
> chain, expiry dates, that the hostname is in the SAN, and revocation status
> (OCSP/CRL). If all pass, trust is established.

### Advanced / Staff

**Q: How would you reduce TLS handshake latency for a global user base?**
> Use TLS 1.3 (1-RTT vs 2-RTT). Enable session resumption / 0-RTT for reconnections
> (with replay protection — only idempotent requests). Enable OCSP stapling to skip
> the CA round trip. Terminate TLS at CDN edge nodes close to users. Use HTTP/2 or
> HTTP/3 to reuse one connection across many requests. Prefer ECDSA certs for faster
> signatures.

**Q: Where would you terminate TLS in a microservices architecture, and why?**
> Depends on the trust model. Edge/LB termination is simplest and lowest latency —
> fine if the internal network is trusted. For zero-trust, re-encrypt to the backend
> or use mTLS between services (service mesh like Istio) so every hop is
> authenticated and encrypted, and no service implicitly trusts another based on
> network position.

**Q: How do you protect against "harvest now, decrypt later" quantum attacks?**
> Adopt hybrid post-quantum key exchange (classical ECDHE + a PQ KEM like ML-KEM),
> so recorded traffic stays secure even against a future quantum computer. Rotate to
> larger/PQ-safe keys, and ensure forward secrecy is on everywhere.

---

## 16. Hands-on Exercises

### Exercise 1 — Read a real certificate

Run this against any HTTPS site and identify the issuer, expiry, and SAN entries:

```bash
openssl s_client -connect github.com:443 -servername github.com </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates -ext subjectAltName
```

<details>
<summary>What to look for</summary>

- `subject` = the domain the cert is for
- `issuer` = which CA signed it (the intermediate)
- `notBefore` / `notAfter` = validity window — is it expiring soon?
- `subjectAltName` = every hostname this cert is valid for
</details>

### Exercise 2 — Explain the output order

A client connects with TLS 1.3 to a server it has never contacted. How many round
trips before the first byte of encrypted HTTP response can arrive?

<details>
<summary>Answer</summary>

**One round trip.** The client sends ClientHello *with* its KeyShare. The server
responds with ServerHello + KeyShare + (encrypted) certificate + Finished, and can
begin sending encrypted application data in that same flight. So after 1-RTT, data
flows. (On a *resumed* connection with a PSK, 0-RTT is possible.)
</details>

### Exercise 3 — Diagnose the bug

Users on some older Android devices get `ERR_CERT_AUTHORITY_INVALID`, but it works
fine in your desktop Chrome. The cert is valid and not expired. What's the likely
cause?

<details>
<summary>Answer</summary>

The server is serving only the **leaf certificate**, not the full chain. Your
desktop browser already has the intermediate cached, so it builds the chain fine.
The older devices don't have it cached and can't reach the root, so validation
fails. **Fix:** configure the server to send `fullchain.pem` (leaf + intermediates).
</details>

### Exercise 4 — Design question

You're designing the connection security for a new banking API consumed by a mobile
app and partner backends. What TLS decisions do you make?

<details>
<summary>Discussion points</summary>

- Enforce TLS 1.3 (min 1.2), modern AEAD ciphers, forward secrecy.
- Use **mTLS** so partner backends authenticate with client certs, not just API keys.
- Consider **certificate pinning** in the mobile app to resist rogue-CA MITM.
- **OCSP stapling** + short-lived certs + automated rotation.
- HSTS with preload; redirect all HTTP → HTTPS.
- Store private keys in an HSM / KMS; never in the app bundle or repo.
- Monitor Certificate Transparency logs for mis-issued certs on your domain.
</details>

---

## 17. Revision & Cheat Sheet

### 5-Minute Summary

- TLS = encryption + integrity + authentication for network traffic. HTTPS = HTTP over TLS.
- SSL is the dead predecessor; "SSL cert" really means a TLS cert.
- **Hybrid crypto:** asymmetric (slow) in the handshake to agree a key, symmetric (fast) for data.
- The **handshake** authenticates the server via its certificate and derives a shared session key via ECDHE.
- **Chain of trust:** leaf cert ← intermediate CA ← root CA (in browser trust store).
- **Forward secrecy** (ECDHE) means stealing the private key later can't decrypt past traffic.
- **TLS 1.3** = 1-RTT (vs 2-RTT), mandatory forward secrecy, weak options removed.
- Top real-world failures: expired certs, missing intermediates, mixed content, leaked private keys.

### Flash Cards

| Q | A |
|---|---|
| SSL vs TLS? | TLS is the modern secure successor; SSL is deprecated |
| Three TLS guarantees? | Confidentiality, integrity, authentication |
| Why both asymmetric + symmetric? | Asymmetric to exchange a key, symmetric for fast data |
| What gives forward secrecy? | Ephemeral Diffie-Hellman (ECDHE) |
| TLS 1.2 vs 1.3 round trips? | 2-RTT vs 1-RTT (0-RTT on resume) |
| What's in the chain of trust? | leaf ← intermediate CA ← root CA (trust store) |
| Field checked for domain match? | Subject Alternative Name (SAN) |
| What is mTLS? | Both client and server present certificates |
| What is OCSP stapling? | Server attaches signed revocation status to handshake |
| Most common TLS outage cause? | Expired certificate |

### Cheat Sheet

```
┌──────────────────────────────────────────────────────────────┐
│                     SSL / TLS — CHEAT SHEET                    │
├──────────────────────────────────────────────────────────────┤
│ GOAL                                                           │
│   Confidentiality (encrypt) + Integrity (tamper-proof)         │
│   + Authentication (verify server identity)                    │
├──────────────────────────────────────────────────────────────┤
│ HYBRID CRYPTO                                                  │
│   Handshake  → asymmetric (RSA/ECDSA + ECDHE)  [slow, once]    │
│   Data        → symmetric (AES-GCM/ChaCha20)    [fast, bulk]   │
├──────────────────────────────────────────────────────────────┤
│ HANDSHAKE (TLS 1.3, 1-RTT)                                     │
│   ClientHello + KeyShare  ─►                                   │
│                           ◄─  ServerHello + KeyShare + Cert    │
│   both derive session key via ECDHE → encrypted data           │
├──────────────────────────────────────────────────────────────┤
│ CHAIN OF TRUST                                                 │
│   leaf cert ← intermediate CA ← root CA (in trust store)       │
│   verify: signature chain + expiry + SAN match + revocation    │
├──────────────────────────────────────────────────────────────┤
│ KEY PROPERTIES                                                 │
│   Forward secrecy   → ECDHE, ephemeral keys, mandatory in 1.3  │
│   Session resumption→ skip handshake on reconnect (PSK/ticket) │
│   0-RTT             → data in first packet (idempotent only)   │
├──────────────────────────────────────────────────────────────┤
│ VERSIONS                                                       │
│   Use TLS 1.3, min 1.2. Disable 1.0/1.1/SSLv3, RC4, MD5, SHA-1 │
├──────────────────────────────────────────────────────────────┤
│ TOP MISTAKES                                                   │
│   Expired cert · missing intermediate · mixed content          │
│   leaked private key · weak ciphers · disabling validation     │
├──────────────────────────────────────────────────────────────┤
│ DEBUG                                                          │
│   openssl s_client -connect host:443 -servername host          │
│   SSL Labs test · DevTools Security tab · testssl.sh           │
└──────────────────────────────────────────────────────────────┘
```

---

## 18. References

- RFC 8446 — The Transport Layer Security (TLS) Protocol Version 1.3
- RFC 5246 — TLS 1.2
- Mozilla SSL Configuration Generator — ssl-config.mozilla.org
- SSL Labs Server Test — ssllabs.com/ssltest
- Let's Encrypt documentation — letsencrypt.org/docs
- "Bulletproof SSL and TLS" by Ivan Ristić
- Cloudflare Learning Center — TLS / SSL articles
- High Performance Browser Networking (Ilya Grigorik) — TLS chapter
