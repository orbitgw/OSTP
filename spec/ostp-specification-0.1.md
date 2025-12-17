---
title: "OSTP (ORBITGW's Secure Transfer Protocol) Specification Version 0.1"
author: "orbitgw"
version: "0.1.2"
origin_publish_timestamp: "1764522431"
copyright: "Copyright (c) 2025 orbitgw <orbitgw@foxmail.com>"
license: "[CC-BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/)"
---

# OSTP (ORBITGW's Secure Transfer Protocol) Specification Version 0.1

## Abstract

This document specifies OSTP (ORBITGW's Secure Transfer Protocol), a secure transport layer protocol designed for reliable data transmission over TCP connections. OSTP provides authentication, confidentiality, and integrity protection using pre-shared keys and AES-256-GCM encryption.

## Status of This Memo

This document specifies an experimental protocol for the community. It does not specify an Internet standard.

## Table of Contents
- [OSTP (ORBITGW's Secure Transfer Protocol) Specification Version 0.1](#ostp-orbitgws-secure-transfer-protocol-specification-version-01)
  - [Abstract](#abstract)
  - [Status of This Memo](#status-of-this-memo)
  - [Table of Contents](#table-of-contents)
  - [1. Introduction](#1-introduction)
  - [2. Requirements Notation](#2-requirements-notation)
  - [3. Terminology](#3-terminology)
  - [4. Protocol Overview](#4-protocol-overview)
  - [5. Key Derivation](#5-key-derivation)
    - [5.1. ASK Derivation](#51-ask-derivation)
    - [5.2. SSK Derivation](#52-ssk-derivation)
  - [6. Message Formats](#6-message-formats)
    - [6.1. Encryption Format](#61-encryption-format)
    - [6.2. Client Handshake Message](#62-client-handshake-message)
    - [6.3. Server Handshake Response (Success)](#63-server-handshake-response-success)
    - [6.4. Server Handshake Response (Failure)](#64-server-handshake-response-failure)
    - [6.5. Data Transfer Message](#65-data-transfer-message)
  - [7. Handshake Protocol](#7-handshake-protocol)
    - [7.1. Client Initialization](#71-client-initialization)
    - [7.2. Server Processing](#72-server-processing)
    - [7.3. Server Response (Success Case)](#73-server-response-success-case)
    - [7.4. Server Response (Failure Case)](#74-server-response-failure-case)
    - [7.5. Client Final Verification](#75-client-final-verification)
  - [8. Data Transfer Protocol](#8-data-transfer-protocol)
  - [9. Security Considerations](#9-security-considerations)
    - [9.1. Key Management](#91-key-management)
    - [9.2. Cryptographic Security](#92-cryptographic-security)
    - [9.3. Protocol Security](#93-protocol-security)
    - [9.4. Implementation Considerations](#94-implementation-considerations)
  - [10. IANA Considerations](#10-iana-considerations)
  - [License and Copyright Notice](#license-and-copyright-notice)

## 1. Introduction

OSTP is a secure transport protocol that establishes authenticated and encrypted communication channels between clients and servers using pre-shared keys. The protocol employs a two-phase handshake mechanism to establish session keys and subsequently protects all data transmission with AES-256-GCM encryption.

## 2. Requirements Notation

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 [RFC2119](https://www.rfc-editor.org/rfc/rfc2119) [RFC8174](https://www.rfc-editor.org/rfc/rfc8174) when, and only when, they appear in all capitals, as shown here.

## 3. Terminology

The following table defines the key terms used in this specification:

| Term | Acronym |     Description |
|:------|:---------|:-------------|
| Pre-shared Secret Key | PSK | A 32-byte symmetric key shared between client and server through secure means prior to communication. The PSK MUST NOT be transmitted over the network and MUST NOT be used directly for encrypting plaintext. |
| Client Random Code | CRC | A randomly generated code by the client. MAY be transmitted in cleartext. |
| Server Random Code | SRC | A randomly generated code by the server. MAY be transmitted in cleartext. |
| Auth Secret Key | ASK | A 32-byte authentication key derived from PSK and CRC using HMAC-SHA256. Used for encrypting the initial handshake message from client. |
| Session Secret Key | SSK | A 32-byte session key derived from PSK and SRC using HMAC-SHA256. Used for all encryption after the server's handshake response until connection termination. |
| Key Identifier | KEY_ID | An identifier transmitted in cleartext that identifies which PSK to use. On the server side, each KEY_ID MUST correspond to exactly one PSK. |

## 4. Protocol Overview

OSTP operates in two distinct phases:

1. **Handshake Phase**: Establishes authenticated session keys
2. **Data Transfer Phase**: Secures all subsequent data transmission

The protocol uses AES-256-GCM for authenticated encryption with additional associated data (AAD).

## 5. Key Derivation

### 5.1. ASK Derivation

The Authentication Secret Key (ASK) SHALL be derived as follows:

```
ASK = HMAC-SHA256(key = PSK, plaintext = CRC)
```

Where:
- PSK is the 32-byte Pre-shared Secret Key
- CRC is the Client Random Code
- The output is exactly 32 bytes

### 5.2. SSK Derivation

The Session Secret Key (SSK) SHALL be derived as follows:

```
SSK = HMAC-SHA256(key = PSK, plaintext = SRC)
```

Where:
- PSK is the 32-byte Pre-shared Secret Key
- SRC is the Server Random Code
- The output is exactly 32 bytes

## 6. Message Formats

### 6.1. Encryption Format

All encryption SHALL use AES-256-GCM with the following parameters:

```
AES-256-GCM(plaintext, key, iv=nonce, aad='OSTP_GENERAL_AAD') 
=> nonce + encryptedData + authTag
```

Where:
- `nonce` is a randomly generated 12-byte value
- `encryptedData` is the ciphertext
- `authTag` is the 16-byte authentication tag
- `aad` is set to the fixed string 'OSTP_GENERAL_AAD'

### 6.2. Client Handshake Message

```
[OSTP <VERSION>] <KEY_ID> <CRC> <ENCRYPTED_AUTH_MSG>
```

Fields:
- `<VERSION>`: Protocol version string (e.g., "1.0")
- `<KEY_ID>`: Key identifier for PSK selection
- `<CRC>`: Client Random Code (16-32 bytes recommended)
- `<ENCRYPTED_AUTH_MSG>`: AES-256-GCM encrypted authentication message using ASK

### 6.3. Server Handshake Response (Success)

```
[OSTP <VERSION>] <SRC> <ENCRYPTED_MSG>
```

Fields:
- `<VERSION>`: Protocol version string
- `<SRC>`: Server Random Code (16-32 bytes recommended)
- `<ENCRYPTED_MSG>`: AES-256-GCM encrypted response message using SSK

### 6.4. Server Handshake Response (Failure)

```
[OSTP <VERSION>] <STATUS_CODE> <FAILED_MSG>
```

Fields:
- `<VERSION>`: Protocol version string
- `<STATUS_CODE>`: Numeric error code
- `<FAILED_MSG>`: Human-readable error message

### 6.5. Data Transfer Message

```
<ENCRYPTED_MSG>
```

Where:
- `<ENCRYPTED_MSG>`: AES-256-GCM encrypted data using SSK

## 7. Handshake Protocol

### 7.1. Client Initialization

1. The client SHALL generate a cryptographically secure random CRC
2. The client SHALL derive ASK using PSK and CRC
3. The client SHALL obtain the current UNIX timestamp
4. The client SHALL form the plaintext: `Hello <HOST> <TIMESTAMP>`
   - `<HOST>` MUST be the server's IP address or domain name
   - `<TIMESTAMP>` MUST be the current UNIX timestamp as a string
5. The client SHALL encrypt this plaintext using ASK to produce ENCRYPTED_AUTH_MSG
6. The client SHALL transmit: `[OSTP <VERSION>] <KEY_ID> <CRC> <ENCRYPTED_AUTH_MSG>`

### 7.2. Server Processing

1. The server SHALL parse the client message and extract KEY_ID, CRC, and ENCRYPTED_AUTH_MSG
2. The server SHALL locate the corresponding PSK using KEY_ID
3. The server SHALL derive ASK using the same method as the client
4. The server SHALL decrypt ENCRYPTED_AUTH_MSG using ASK
5. The server SHALL verify:
   - The decrypted text matches the pattern `Hello <HOST> <TIMESTAMP>`
   - The `<HOST>` value matches the server's expected identity
   - The `<TIMESTAMP>` is within an acceptable time window (recommended: ±5 minutes from server time)

### 7.3. Server Response (Success Case)

1. The server SHALL generate a cryptographically secure random SRC
2. The server SHALL derive SSK using PSK and SRC
3. The server SHALL obtain the current UNIX timestamp
4. The server SHALL form the plaintext: `Hello Client from <HOST> <TIMESTAMP>`
5. The server SHALL encrypt this plaintext using SSK to produce ENCRYPTED_MSG
6. The server SHALL transmit: `[OSTP <VERSION>] <SRC> <ENCRYPTED_MSG>`

### 7.4. Server Response (Failure Case)

If authentication fails at any point, the server SHALL transmit:
`[OSTP <VERSION>] <STATUS_CODE> <FAILED_MSG>`

Recommended status codes:

- 0x01: Authentication Failed
- 0x02: Invalid Request
- 0x03: Timestamp Out of Range
- 0x04: Internal Server Error

### 7.5. Client Final Verification

1. Upon receiving successful response, the client SHALL derive SSK using PSK and SRC
2. The client SHALL decrypt ENCRYPTED_MSG using SSK
3. The client SHALL verify:
   - The decrypted text matches the pattern `Hello Client from <HOST> <TIMESTAMP>`
   - The `<TIMESTAMP>` is within an acceptable time window (recommended: ±5 minutes from client time)

## 8. Data Transfer Protocol

After successful handshake completion:

1. Both client and server SHALL use SSK for all subsequent encryption
2. All messages SHALL be encrypted as: `<ENCRYPTED_MSG>`
3. The same AES-256-GCM parameters SHALL be used as specified in Section 6.1

## 9. Security Considerations

### 9.1. Key Management

- PSKs MUST be securely distributed and stored
- PSKs MUST be 32 bytes in length
- KEY_ID values SHOULD not reveal information about the associated PSK

### 9.2. Cryptographic Security

- CRC and SRC MUST be generated using cryptographically secure random number generators
- Nonces for AES-GCM MUST be unique for each encryption operation under the same key
- The same nonce MUST NEVER be reused with the same key
- Implementations MUST verify authentication tags before processing ciphertext

### 9.3. Protocol Security

- The handshake protocol provides mutual authentication through proof of PSK knowledge
- Timestamp validation provides protection against replay attacks during handshake
- The timestamp window SHOULD be configurable based on security requirements
- Forward secrecy is NOT provided by this protocol

### 9.4. Implementation Considerations

- Implementations SHOULD include rate limiting to prevent brute force attacks
- Failed authentication attempts SHOULD be logged
- Connections SHOULD be terminated after multiple authentication failures

## 10. IANA Considerations

This document has no IANA actions.

## License and Copyright Notice

Copyright (c) 2025 orbitgw <orbitgw@foxmail.com> 

This document is authorized under the [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/) license.