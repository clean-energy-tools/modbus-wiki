---
title: Summary of MODBUS/TCP Security
Summary: MODBUS/TCP Security specification defining secure communication over TLS, mutual authentication, role-based authorization, and certificate management.
Sources:
  - raw/MODBUS/modbussecurityprotocol.md
Categories:
  - specifications
  - security
  - tls
type: summary
date-created: 2026-04-18T12:00:00+03:00
last-updated: 2026-04-18T14:43:52+03:00
---

This document defines the MODBUS/TCP Security protocol specification that adds TLS encryption and authentication to MODBUS/TCP communications (source: [modbussecurityprotocol.md](/raw/MODBUS/modbussecurityprotocol.md)).

## Protocol Overview

[MODBUS TCP Security](/wiki/concepts/modbus-tcp-security.md) (MBAPS) encapsulates standard MODBUS/TCP within TLS, providing secure, authenticated communication for MODBUS devices over TCP/IP networks (source: [modbussecurityprotocol.md](/raw/MODBUS/modbussecurityprotocol.md)).

### Key Characteristics

| Property | Value |
|----------|-------|
| Port | 802 (secure MODBUS) |
| TLS Version | 1.2 minimum, 1.3 recommended |
| Authentication | Mutual TLS (client and server certificates) |
| Certificate Format | x.509v3 |
| Role Encoding | Certificate extension (OID 1.3.6.1.4.1.50316.802.1) |

## Security Architecture

### Mutual Authentication

Both client and server must authenticate using X.509v3 certificates:

**Client Authentication:**
- Client presents certificate to server during TLS handshake
- Server validates client certificate
- Server extracts role from certificate extension
- Server applies authorization rules based on role

**Server Authentication:**
- Server presents certificate to client during TLS handshake
- Client validates server certificate
- Client verifies server identity

### Role-Based Authorization

Roles are encoded in x.509v3 certificate extensions to implement access control:

| Property | Value |
|-----------|-------|
| OID | 1.3.6.1.4.1.50316.802.1 |
| Encoding | ASN1 UTF8String |
| Usage | Server extracts role, applies authorization rules |

**Authorization Rules:**
- Roles defined by vendor/implementation
- Server enforces role-based permissions
- Unauthorized requests receive exception code 0x01 (Illegal Function)
- Role mapping to permissions is vendor-specific

## TLS Configuration

### Required Cipher Suite

All implementations must support:

```
TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
```

This cipher suite provides:
- **ECDHE**: Elliptic Curve Diffie-Hellman Ephemeral (perfect forward secrecy)
- **RSA**: RSA authentication
- **AES-128-GCM**: AES encryption in Galois/Counter Mode
- **SHA256**: SHA-256 for integrity

### TLS Version Requirements

| Version | Status |
|---------|--------|
| TLS 1.0 | Not supported |
| TLS 1.1 | Not supported |
| TLS 1.2 | Minimum required |
| TLS 1.3 | Recommended |

## Protocol Stack

### Secure MODBUS Frame Structure

```
[TLS Record Header][Encrypted MBAP Header][Encrypted MODBUS PDU][TLS MAC]
```

**Original MODBUS/TCP ADU:**
```
[MBAP Header: 7 bytes][MODBUS PDU: up to 253 bytes]
```

**Secure MODBUS/TCP ADU:**
```
[TLS Handshake/Record Layer]
  └─ [MBAP Header: 7 bytes][MODBUS PDU: up to 253 bytes]
```

The MBAP header and PDU are identical to standard MODBUS/TCP, just encrypted within TLS.

### Port Allocation

| Protocol | Port | Notes |
|----------|------|-------|
| MODBUS/TCP | 502 | Standard unencrypted MODBUS |
| MODBUS/TCP Security | 802 | Secure MODBUS over TLS |

Both ports may be simultaneously available on the same device.

## Certificate Management

### Certificate Requirements

**Client Certificate:**
- Must be valid X.509v3 certificate
- Contains role extension (OID 1.3.6.1.4.1.50316.802.1)
- Signed by trusted CA or self-signed for private networks
- Client authentication enabled in TLS configuration

**Server Certificate:**
- Must be valid X.509v3 certificate
- Contains server name/IP in Subject or SAN
- Signed by trusted CA or self-signed for private networks
- Server authentication enabled in TLS configuration

### Role Extension Format

The role extension is encoded in the certificate as:

```
Extension OID: 1.3.6.1.4.1.50316.802.1
Encoding: UTF8String
Value: [role-name]
```

Example roles:
- `operator` - Basic operational access
- `administrator` - Full configuration access
- `monitor` - Read-only access
- `maintenance` - Maintenance operations only

### Certificate Validation

**Server Validation:**
- Verify certificate chain to trusted root
- Check certificate validity period
- Verify role extension present and valid
- Extract role for authorization

**Client Validation:**
- Verify certificate chain to trusted root
- Check certificate validity period
- Verify server name/IP matches certificate
- Optional: Verify server role (if applicable)

## Authorization Model

### Access Control

Server enforces role-based access control based on extracted client role:

| Operation | Operator | Administrator | Monitor |
|-----------|----------|---------------|---------|
| Read Coils | ✓ | ✓ | ✓ |
| Read Registers | ✓ | ✓ | ✓ |
| Write Coils | ✓ | ✓ | ✗ |
| Write Registers | ✓ | ✓ | ✗ |
| Configure Device | ✗ | ✓ | ✗ |

*Note: Specific permissions are vendor/implementation-defined*

### Exception Handling

Unauthorized requests receive standard MODBUS exception response:

| Exception Code | Hex | Name |
|---------------|-----|------|
| 1 | 0x01 | ILLEGAL FUNCTION |

The server returns exception code 0x01 when the client role does not have permission to execute the requested function code.

## Connection Establishment

### Secure Connection Handshake

1. **TCP Connection:** Client connects to server port 802
2. **TLS Handshake:**
   - Client presents certificate
   - Server presents certificate
   - Mutual authentication performed
   - Cipher suite negotiated
   - TLS session established
3. **MODBUS Communication:** Standard MODBUS/TCP exchanged over encrypted channel
4. **Connection Termination:** TLS shutdown and TCP close

### Session Management

- Keep TLS session alive for multiple MODBUS transactions
- Use TLS session resumption for performance
- Implement connection management same as MODBUS/TCP
- Use SO_KEEPALIVE to detect dead connections

## Security Considerations

### Best Practices

**Certificate Management:**
- Use certificates from trusted Certificate Authority for production
- Implement certificate revocation checking (CRL/OCSP)
- Rotate certificates periodically
- Protect private keys with appropriate access controls

**TLS Configuration:**
- Use TLS 1.3 when available
- Disable weak cipher suites
- Disable TLS compression
- Enable perfect forward secrecy
- Implement certificate pinning in security-critical environments

**Access Control:**
- Define clear role definitions and permissions
- Implement least privilege principle
- Log authorization failures for audit
- Monitor for unauthorized access attempts

### Security Benefits

**Confidentiality:**
- All MODBUS traffic encrypted
- Data values protected from eavesdropping
- Device state information protected

**Integrity:**
- TLS MAC ensures data integrity
- Detects tampering in transit
- Prevents man-in-the-middle attacks

**Authentication:**
- Mutual authentication of client and server
- Prevents unauthorized device access
- Role-based authorization for fine-grained control

## Comparison: Standard vs Secure MODBUS/TCP

| Characteristic | MODBUS/TCP | MODBUS/TCP Security |
|---------------|-------------|----------------------|
| Port | 502 | 802 |
| Encryption | None | TLS 1.2+ |
| Authentication | None | Mutual TLS |
| Authorization | None | Role-based |
| Cipher Suite | N/A | TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256 |
| Certificate | None | x.509v3 |
| Forward Secrecy | No | Yes (if using ECDHE) |

## Related pages

- [MODBUS TCP Security](/wiki/concepts/modbus-tcp-security.md)
- [MODBUS TCP](/wiki/concepts/modbus-tcp.md)
- [tls](/wiki/concepts/tls.md)
- [mutual-authentication](/wiki/concepts/mutual-authentication.md)
- [role-based-authorization](/wiki/concepts/role-based-authorization.md)

## Backlinks

No backlinks (yet)

------------

MODBUS is a trademark of the Modbus Organization, Inc.

All information on this page is derived by summarizing and analyzing solely the MODBUS specifications published by the Modbus Organization, Inc.  The specifications are copyright by the Modbus Organization.

By deriving this information solely from the specifications we hope to stay true to the specification.

No infringement is intended.
