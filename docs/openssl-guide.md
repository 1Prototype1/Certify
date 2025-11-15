# OpenSSL Certificate Generation Guide

This guide covers generating a local Root CA and issuing a server certificate for local development or internal use.

## Option A: ECC (ECDSA) Certificates

### 1. Generate the Root CA private key (ECC)
Generate an ECC private key using the secp384r1 curve:
```bash
openssl ecparam -name secp384r1 -genkey -out rootCA.key
```

Encrypt the key with AES-256 (you will be prompted for a passphrase):
```bash
openssl ec -in rootCA.key -aes256 -out rootCA.key.enc
```

### 2. Create the self-signed Root CA certificate (ECC)
Common Name (CN) = your domain (e.g., example.com), not a host or wildcard.
```bash
openssl req -new -x509 -sha384 -days 3650 -key rootCA.key.enc -out rootCA.crt
```
This produces `rootCA.crt` (valid ~10 years).

### 3. Create server private key and CSR (ECC)
Generate the server private key:
```bash
openssl ecparam -name secp384r1 -genkey -out server.key
```

Create the Certificate Signing Request (CN = hostname, e.g., www.example.com):
```bash
openssl req -new -sha384 -key server.key -out server.csr
```

### 4. Create certificate extensions file
File: `cert.ext` (must include at least the required basic constraint):
```
basicConstraints=CA:FALSE
authorityKeyIdentifier=keyid,issuer
subjectKeyIdentifier=hash
keyUsage = digitalSignature, nonRepudiation, keyAgreement, dataEncipherment
extendedKeyUsage = clientAuth
subjectAltName = DNS:localhost, DNS:example.com
```
Note: For ECC, use `keyAgreement` instead of `keyEncipherment`.

### 5. Sign the server certificate with the Root CA (ECC)
```bash
openssl x509 -req -in server.csr -CA rootCA.crt -CAkey rootCA.key.enc -extfile cert.ext -days 1825 -sha384 -out server.crt
```
Outputs:
- `server.crt` (server certificate, ~5 years)

### Summary of ECC commands
```bash
openssl ecparam -name secp384r1 -genkey -out rootCA.key
openssl ec -in rootCA.key -aes256 -out rootCA.key.enc
openssl req -new -x509 -sha384 -days 3650 -key rootCA.key.enc -out rootCA.crt
openssl ecparam -name secp384r1 -genkey -out server.key
openssl req -new -sha384 -key server.key -out server.csr
openssl x509 -req -in server.csr -CA rootCA.crt -CAkey rootCA.key.enc -extfile cert.ext -days 1825 -sha384 -out server.crt
```

---

## Option B: RSA Certificates

### 1. Generate the Root CA private key (RSA)
Encrypted (you will be prompted for a passphrase):
```bash
openssl genrsa -aes256 -out rootca.key 4096
```
(Use `-aes256` to protect the key; remove it for an unencrypted key.)

### 2. Create the self-signed Root CA certificate (RSA)
Common Name (CN) = your domain (e.g., example.com), not a host or wildcard.
```bash
openssl req -x509 -new -nodes -key rootca.key -sha256 -days 1826 -out rootca.crt
```
This produces `rootca.crt` (valid ~5 years).

### 3. Create server private key and CSR (RSA)
Common Name (CN) = hostname (e.g., www.example.com).
```bash
openssl req -new -nodes -out cert.csr -newkey rsa:4096 -keyout cert.key
```

### 4. Create certificate extensions file (RSA)
File: `cert.v3.ext` (must include at least the required basic constraint):
```
basicConstraints=CA:FALSE
authorityKeyIdentifier=keyid,issuer
subjectKeyIdentifier=hash
keyUsage = digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = DNS:localhost, DNS:example.com
```
Adjust SAN entries as needed.

### 5. Sign the server certificate with the Root CA (RSA)
```bash
openssl x509 -req -in cert.csr \
    -CA rootca.crt -CAkey rootca.key -CAcreateserial \
    -out cert.crt -days 730 -sha256 -extfile ./cert.v3.ext
```
Outputs:
- `cert.crt` (server certificate, ~2 years)
- `rootca.srl` (serial tracking file)

### Summary of RSA commands
```bash
openssl genrsa -aes256 -out rootca.key 4096
openssl req -x509 -new -nodes -key rootca.key -sha256 -days 1826 -out rootca.crt
openssl req -new -nodes -out cert.csr -newkey rsa:4096 -keyout cert.key
openssl x509 -req -in cert.csr -CA rootca.crt -CAkey rootca.key -CAcreateserial -out cert.crt -days 730 -sha256 -extfile ./cert.v3.ext
```

---

## 6. Verify artifacts
```bash
openssl x509 -in cert.crt -noout -text
openssl verify -CAfile rootca.crt cert.crt
```

## 7. (Optional) Create a full chain file
Some servers expect a bundle:
```bash
cat cert.crt rootca.crt > fullchain.crt
```

## 8. Trust the Root CA (local development)
- macOS: Keychain Access > System > Certificates > import `rootca.crt` > mark as trusted.
- Linux: Copy to `/usr/local/share/ca-certificates/` then `sudo update-ca-certificates`.
- Windows: Manage Computer Certificates > Trusted Root Certification Authorities > Certificates > Import.

## 9. File permissions
```bash
chmod 400 rootca.key cert.key
```

## 10. Renewal
Regenerate only the server CSR + signed cert before expiry. Keep `rootca.key` (or `rootCA.key.enc`) secure; compromise requires re-issuing all certs.

## Cleanup (optional)
Remove CSR after signing:
```bash
shred -u cert.csr  # or server.csr for ECC
```

## Notes on ECC vs RSA
- **ECC (ECDSA)**: Smaller keys (384-bit ≈ 7680-bit RSA), faster signing/verification, lower bandwidth. Modern standard.
- **RSA**: Wider compatibility with legacy systems. Larger keys and certificates.
- Both are secure when properly configured. Choose based on compatibility requirements.
