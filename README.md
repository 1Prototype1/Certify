# Certify

Lightweight toolkit for:
1. Generating local CA + server certificates with OpenSSL for development / internal use.
2. Managing Cloudflare Origin TLS Client Authentication certificates (upload, list, enable per-hostname).

## Documentation

| Topic | Description |
|-------|-------------|
| [OpenSSL Guide](./docs/openssl-guide.md) | Step-by-step commands to create a Root CA, issue server certs, verify, trust locally, and renew. |
| [Cloudflare Wrapper](./docs/cloudflare-wrapper.md) | Async Python client for Origin TLS Client Auth: upload certs, list, and enable AOP for hostnames. |
| [Nginx SSL Client Auth](./docs/nginx-sslclient.md) | Guide to configuring Nginx for client certificate authentication using generated certs. |

## Quick Start

### OpenSSL (TL;DR)
#### ECC (ECDSA)
```
openssl ecparam -name secp384r1 -genkey -out rootCA.key
openssl ec -in rootCA.key -aes256 -out rootCA.key.enc
openssl req -new -x509 -sha384 -days 3650 -key rootCA.key.enc -out rootCA.crt
openssl ecparam -name secp384r1 -genkey -out server.key
openssl req -new -sha384 -key server.key -out server.csr
openssl x509 -req -in server.csr -CA rootCA.crt -CAkey rootCA.key.enc -extfile cert.ext -days 1825 -sha384 -out server.crt
```
#### RSA
```
openssl genrsa -aes256 -out rootca.key 4096
openssl req -x509 -new -nodes -key rootca.key -sha256 -days 1826 -out rootca.crt
openssl req -new -nodes -out cert.csr -newkey rsa:4096 -keyout cert.key
openssl x509 -req -in cert.csr -CA rootca.crt -CAkey rootca.key -CAcreateserial -out cert.crt -days 730 -sha256 -extfile ./cert.v3.ext
```
See the full [OpenSSL Guide](./docs/openssl-guide.md) for SANs, verification, trust, and renewal.

### Cloudflare Wrapper (Snippet)
```python
import asyncio
from cloudflare import Cloudflare

cf = Cloudflare("<zone-id>", "<api-token>")

async def run():
    upload = await cf.upload_cert("pki/cert.crt", "pki/cert.key")
    certs = await cf.get_certs()
    cert_id = upload.get("result", {}).get("id")
    if cert_id:
        await cf.enable_aop("example.com", cert_id, True)
    print(certs)

asyncio.run(run())
```
More in the [Cloudflare Wrapper docs](./docs/cloudflare-wrapper.md).

## Requirements
- Python 3.11+
- `aiohttp`
- OpenSSL installed (CLI) for certificate generation

## Security Notes
- Keep `rootca.key` and private keys secret; never commit them.
- Use distinct API tokens with least privilege.
- Consider adding structured error handling & retries for production.

---
For detailed workflows, open the linked docs above.