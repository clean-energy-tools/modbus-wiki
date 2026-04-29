---
title: MODBUS TCP Security
Summary: MODBUS/TCP Security (MBAPS) protocol adding TLS encryption, mutual authentication, and role-based authorization to MODBUS/TCP.
Sources:
  - raw/MODBUS/modbussecurityprotocol.md
Categories:
  - security
  - tls
  - authentication
type: concept
date-created: 2026-04-18T12:00:00+03:00
last-updated: 2026-04-18T14:43:24+03:00
---

MODBUS/TCP Security (MBAPS) is MODBUS/TCP wrapped in encryption (TLS), making messages private and verifying who's talking (source: [modbussecurityprotocol.md](/raw/MODBUS/modbussecurityprotocol.md)). Think of it like putting MODBUS messages in a locked, tamper-proof envelope.

## What MODBUS TCP Security Does

MODBUS/TCP Security adds encryption and authentication to MODBUS/TCP using TLS version 1.2 or newer (source: [modbussecurityprotocol.md](/raw/MODBUS/modbussecurityprotocol.md)). This keeps messages private, prevents tampering, and proves both sides are who they claim to be.

### Security Features

| Feature | Details |
|----------|-------|
| Port number | 802 (different from regular MODBUS on 502) |
| Encryption version | TLS 1.2 minimum, TLS 1.3 better |
| Who can connect | Both client and server verify each other with certificates |
| Certificate type | x.509v3 (standard digital certificates) |
| User roles | Stored in certificate (special field) |
| Encryption method | TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256 |

## How the Encryption Works

### Secure Message Structure

```
[TLS Encryption Wrapper]
  └─ [Encrypted Header: 7 bytes][Encrypted MODBUS Message: up to 253 bytes]
```

**Regular MODBUS/TCP (no encryption):**
```
[Header: 7 bytes][MODBUS Message: up to 253 bytes]
```

**Secure MODBUS/TCP (encrypted):**
- Same header and message format inside
- Everything wrapped in TLS encryption
- Extra authentication code to prevent tampering

## Port Allocation

| Protocol | Port | Notes |
|----------|------|-------|
| MODBUS/TCP | 502 | Standard unencrypted MODBUS |
| MODBUS/TCP Security | 802 | Secure MODBUS over TLS |

Both ports may be simultaneously available on the same device.

## TLS Configuration

### Required Cipher Suite

All implementations must support:

```
TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
```

**Cipher suite components:**
- **ECDHE**: Elliptic Curve Diffie-Hellman Ephemeral - provides perfect forward secrecy
- **RSA**: RSA authentication - authenticates certificates
- **AES-128-GCM**: AES encryption in Galois/Counter Mode - encrypts data
- **SHA256**: SHA-256 for integrity - ensures data integrity

### TLS Version Requirements

| Version | Status | Reason |
|---------|--------|--------|
| TLS 1.0 | Not supported | Security vulnerabilities |
| TLS 1.1 | Not supported | Security vulnerabilities |
| TLS 1.2 | Minimum required | Adequate security |
| TLS 1.3 | Recommended | Best security and performance |

## Security Architecture

### Mutual Authentication

Both client and server must authenticate using X.509v3 certificates (source: [modbussecurityprotocol.md](/raw/MODBUS/modbussecurityprotocol.md)).

**Client Authentication:**
1. Client presents certificate to server during TLS handshake
2. Server validates client certificate chain
3. Server extracts role from certificate extension
4. Server applies authorization rules based on role

**Server Authentication:**
1. Server presents certificate to client during TLS handshake
2. Client validates server certificate chain
3. Client verifies server identity

**Certificate Validation:**
- Verify certificate chain to trusted root
- Check certificate validity period (not before/after dates)
- Verify certificate revocation (CRL/OCSP) - optional but recommended
- Verify role extension present and valid

### Role-Based Authorization

Roles are encoded in x.509v3 certificate extensions to implement access control (source: [modbussecurityprotocol.md](/raw/MODBUS/modbussecurityprotocol.md)).

**Role Extension Format:**
```
Extension OID: 1.3.6.1.4.1.50316.802.1
Encoding: UTF8String
Value: [role-name]
```

**Example Roles:**
- `operator` - Basic operational access
- `administrator` - Full configuration access
- `monitor` - Read-only access
- `maintenance` - Maintenance operations only
- `readonly` - Read-only monitoring

**Authorization Process:**
1. Server extracts role from client certificate
2. Server maps role to permissions
3. Server validates request against role permissions
4. Unauthorized requests receive exception code 0x01 (ILLEGAL FUNCTION)

**Example Authorization Matrix:**

| Operation | Operator | Administrator | Monitor | Readonly |
|-----------|----------|---------------|---------|----------|
| Read Coils | ✓ | ✓ | ✓ | ✓ |
| Read Registers | ✓ | ✓ | ✓ | ✓ |
| Write Coils | ✓ | ✓ | ✗ | ✗ |
| Write Registers | ✓ | ✓ | ✗ | ✗ |
| Configure Device | ✗ | ✓ | ✗ | ✗ |
| Change Settings | ✗ | ✓ | ✗ | ✗ |

*Note: Specific permissions are vendor/implementation-defined*

### Exception Handling for Unauthorized Access

Unauthorized requests receive standard MODBUS exception response (source: [modbussecurityprotocol.md](/raw/MODBUS/modbussecurityprotocol.md)):

| Exception Code | Hex | Name |
|---------------|-----|------|
| 1 | 0x01 | ILLEGAL FUNCTION |

The server returns exception code 0x01 when the client role does not have permission to execute the requested function code.

## Certificate Management

### Certificate Requirements

**Client Certificate:**
- Must be valid X.509v3 certificate
- Contains role extension (OID 1.3.6.1.4.1.50316.802.1)
- Signed by trusted CA or self-signed for private networks
- Client authentication enabled in TLS configuration

**Server Certificate:**
- Must be valid X.509v3 certificate
- Contains server name/IP in Subject or Subject Alternative Name (SAN)
- Signed by trusted CA or self-signed for private networks
- Server authentication enabled in TLS configuration

### Role Extension in Certificates

**Example Certificate Extension:**
```
X509v3 Extension:
  OID: 1.3.6.1.4.1.50316.802.1
  Critical: No
  Value: UTF8String
  Data: "operator"
```

### Certificate Validation

**Server Validation:**
1. Verify certificate chain to trusted root CA
2. Check certificate validity period
3. Verify role extension present and valid
4. Extract role for authorization
5. Optionally check certificate revocation (CRL/OCSP)

**Client Validation:**
1. Verify certificate chain to trusted root CA
2. Check certificate validity period
3. Verify server name/IP matches certificate (Subject or SAN)
4. Optionally check server role (if applicable)
5. Optionally check certificate revocation

## Connection Establishment

### Secure Connection Handshake

1. **TCP Connection:**
   - Client connects to server port 802
   - TCP 3-way handshake completes

2. **TLS Handshake:**
   - Client sends ClientHello with supported cipher suites
   - Server responds with ServerHello, certificate, and ServerKeyExchange
   - Client validates server certificate
   - Client sends certificate and ClientKeyExchange
   - Server validates client certificate
   - Both derive shared secret
   - TLS session established

3. **MODBUS Communication:**
   - Standard MODBUS/TCP exchanged over encrypted channel
   - MBAP header and PDU encrypted
   - Authorization applied based on extracted role

4. **Connection Termination:**
   - TLS shutdown
   - TCP close

### Session Management

- Keep TLS session alive for multiple MODBUS transactions
- Use TLS session resumption for performance
- Implement connection management same as MODBUS/TCP
- Use SO_KEEPALIVE to detect dead connections

## Security Benefits

### Confidentiality

- All MODBUS traffic encrypted with AES-128-GCM
- Data values protected from eavesdropping
- Device state information protected
- Configuration data encrypted

### Integrity

- TLS MAC ensures data integrity
- Detects tampering in transit
- Prevents man-in-the-middle attacks
- Verifies data hasn't been modified

### Authentication

- Mutual authentication of client and server
- Prevents unauthorized device access
- Role-based authorization for fine-grained control
- Certificate-based identity verification

### Authorization

- Role-based access control
- Granular permission assignment
- Centralized access management
- Audit trail of access attempts

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
| Performance | Higher | Lower (TLS overhead) |
| Complexity | Simple | Moderate |

## Security Considerations

### Best Practices

**Certificate Management:**
- Use certificates from trusted Certificate Authority for production
- Implement certificate revocation checking (CRL/OCSP)
- Rotate certificates periodically (every 1-2 years)
- Protect private keys with appropriate access controls
- Use separate certificates for client and server roles

**TLS Configuration:**
- Use TLS 1.3 when available (better security and performance)
- Disable weak cipher suites (only support required cipher)
- Disable TLS compression (CRIME attack prevention)
- Enable perfect forward secrecy (ECDHE)
- Implement certificate pinning in security-critical environments

**Access Control:**
- Define clear role definitions and permissions
- Implement least privilege principle
- Log authorization failures for audit
- Monitor for unauthorized access attempts
- Regularly review and update permissions

### Security Risks

**Without Secure MODBUS/TCP:**
- Plain-text MODBUS traffic visible on network
- No authentication of devices
- No authorization of operations
- Susceptible to man-in-the-middle attacks
- Susceptible to replay attacks

**With Secure MODBUS/TCP:**
- Properly configured, provides strong security
- Weak certificate management undermines security
- Misconfigured roles can allow unauthorized access
- TLS vulnerabilities can affect security

## Implementation Considerations

### TLS Library Selection

Common TLS libraries for MODBUS/TCP Security:
- **OpenSSL** - Most widely used
- **mbedTLS** - Lightweight, suitable for embedded
- **GnuTLS** - Feature-rich
- **WolfSSL** - Embedded-friendly
- **BoringSSL** - Google's fork of OpenSSL

### Performance Considerations

- TLS adds computational overhead
- AES-128-GCM provides good performance/security balance
- Session resumption reduces handshake overhead
- Hardware acceleration (AES-NI) improves performance
- Trade-off: Security vs Performance

### Migration Path

**Phase 1:** Implement alongside standard MODBUS/TCP (both ports 502 and 802)
**Phase 2:** Configure clients to prefer secure port 802
**Phase 3:** Migrate all clients to secure connections
**Phase 4:** Optionally disable insecure port 502 (after full migration)

## Related pages

- [MODBUS TCP](/wiki/concepts/modbus-tcp.md)
- [tls](/wiki/concepts/tls.md)
- [mutual-authentication](/wiki/concepts/mutual-authentication.md)
- [role-based-authorization](/wiki/concepts/role-based-authorization.md)
- [x509-certificates](/wiki/concepts/x509-certificates.md)

## Backlinks

- [Connecting with MODBUS TCP Security](/wiki/answers/connecting-with-modbus-tcp-security.md)
- [TLS and MODBUS Security](/wiki/answers/tls-and-modbus-security.md)
- [What is MBAP](/wiki/answers/what-is-mbap.md)
- [Wireshark MODBUS Analysis](/wiki/answers/wireshark-modbus-analysis.md)
- [Implementation](/wiki/concepts/implementation.md)
- [MODBUS](/wiki/concepts/modbus.md)
- [MODBUS TCP](/wiki/concepts/modbus-tcp.md)
- [MODBUS Usage and Applications](/wiki/concepts/modbus-usage-and-applications.md)
- [Protocol](/wiki/concepts/protocol.md)
- [Summary of MODBUS](/wiki/summaries/MODBUS/MODBUS.md)
- [Summary of modbusprotocolspecification](/wiki/summaries/MODBUS/modbusprotocolspecification.md)
- [Summary of modbussecurityprotocol](/wiki/summaries/MODBUS/modbussecurityprotocol.md)

------------

MODBUS is a trademark of the Modbus Organization, Inc.

All information on this page is derived by summarizing and analyzing solely the MODBUS specifications published by the Modbus Organization, Inc.  The specifications are copyright by the Modbus Organization.

By deriving this information solely from the specifications we hope to stay true to the specification.

No infringement is intended.
