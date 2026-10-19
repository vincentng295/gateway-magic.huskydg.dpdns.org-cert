# gateway-magic certificate

Opened certificate for anyone to use `[*.]gateway-magic.huskydg.io.vn` domain without permissions, for testing purpose.

A free [Let's Encrypt](https://letsencrypt.org/) certificate for `gateway-magic.huskydg.io.vn` and `*.gateway-magic.huskydg.io.vn`, issued and renewed automatically by GitHub Actions, and published **together with its private key** as a GitHub Release so anybody can download and use it.

> [!WARNING]
> **The private key is public. Anyone can impersonate these hostnames.**
> Use this certificate for **testing / personal / lab networks only**. Never use it to protect passwords, tokens, personal data, or anything you care about. It may be revoked or replaced at any time without notice.

## Table of contents

- [Purpose: HTTPS inside the Magic V2Ray LAN](#purpose-https-inside-the-magic-v2ray-lan)
- [What is covered](#what-is-covered)
- [Download](#download)
- [Quick start](#quick-start)
- [Verify the files](#verify-the-files)
- [Keeping your copy fresh](#keeping-your-copy-fresh)
- [How it works](#how-it-works)
- [Run your own copy](#run-your-own-copy)
- [FAQ and troubleshooting](#faq-and-troubleshooting)
- [Disclaimer](#disclaimer)

## Purpose: HTTPS inside the Magic V2Ray LAN

This domain is meant to be used with **Magic V2Ray**, so you can reach services inside the Magic V2Ray LAN over HTTPS without certificate warnings.

- When Magic V2Ray is enabled, the device that shares the hotspot is reachable at **`198.18.0.1`** (IPv6: `fdfe:dcba:9876::1`) inside its network.
- Magic V2Ray's own DNS maps `gateway-magic.huskydg.io.vn` and its subdomains to those addresses (through its DNS hosts), so the name is only expected to work while Magic V2Ray is running. It is not meant to resolve on the public internet.
- A service listening on that device can use this certificate, and clients simply open `https://gateway-magic.huskydg.io.vn/` or `https://<name>.gateway-magic.huskydg.io.vn/` in the browser.
- `198.18.0.0/15` is a range reserved for benchmarking (RFC 2544) and is not routed on the public internet. The address is only meaningful inside the network of the device you are connected to.

If your Magic V2Ray build does not map the name yet, add this to its custom DNS hosts (JSON format):

```json
{
  "domain:gateway-magic.huskydg.io.vn": ["198.18.0.1", "fdfe:dcba:9876::1"]
}
```

## What is covered

| Name | Covered |
|------|---------|
| `gateway-magic.huskydg.io.vn` | ✅ |
| `anything.gateway-magic.huskydg.io.vn` | ✅ (wildcard) |
| `a.b.gateway-magic.huskydg.io.vn` | ❌ (a wildcard covers **one** label only) |
| `huskydg.io.vn`, other subdomains | ❌ |
| IP addresses (`198.18.0.1`, ...) | ❌ (you must open the site by its hostname) |

The names are not meant to resolve on the public internet (see [Purpose](#purpose-https-inside-the-magic-v2ray-lan)). The certificate is validated with the DNS-01 challenge, so it does not depend on where the name points.

## Download

The links never change; the files behind them are replaced on every renewal.

| File | Purpose | Link |
|------|---------|------|
| `fullchain.pem` | **Certificate + intermediate chain. Use this one as the server certificate.** | [download](../../releases/download/cert/fullchain.pem) |
| `privkey.pem` | Private key (public, see warning above) | [download](../../releases/download/cert/privkey.pem) |
| `cert.pem` | Leaf certificate only (no chain, most servers do not want this) | [download](../../releases/download/cert/cert.pem) |
| `chain.pem` | Intermediate chain only | [download](../../releases/download/cert/chain.pem) |
| `gateway-magic-cert.zip` | All of the above | [download](../../releases/download/cert/gateway-magic-cert.zip) |

Direct URL pattern, replace `OWNER/REPO` with this repository:

```text
https://github.com/OWNER/REPO/releases/download/cert/fullchain.pem
https://github.com/OWNER/REPO/releases/download/cert/privkey.pem
```

The release notes show the last update time and the expiry date.

## Quick start

```bash
curl -fL -o fullchain.pem https://github.com/OWNER/REPO/releases/download/cert/fullchain.pem
curl -fL -o privkey.pem   https://github.com/OWNER/REPO/releases/download/cert/privkey.pem
chmod 600 privkey.pem
```

Always pass `fullchain.pem` as the certificate. Passing only `cert.pem` makes some clients fail with "unable to get local issuer certificate".

### nginx

```nginx
server {
    listen 443 ssl;
    server_name gateway-magic.huskydg.io.vn *.gateway-magic.huskydg.io.vn;

    ssl_certificate     /etc/ssl/gateway-magic/fullchain.pem;
    ssl_certificate_key /etc/ssl/gateway-magic/privkey.pem;
}
```

### Caddy

```caddyfile
gateway-magic.huskydg.io.vn, *.gateway-magic.huskydg.io.vn {
    tls /etc/ssl/gateway-magic/fullchain.pem /etc/ssl/gateway-magic/privkey.pem
    reverse_proxy 127.0.0.1:8080
}
```

### Python

```python
import ssl

ctx = ssl.SSLContext(ssl.PROTOCOL_TLS_SERVER)
ctx.load_cert_chain("fullchain.pem", "privkey.pem")
# pass ctx to your server, for example Flask/Werkzeug: app.run(ssl_context=ctx)
```

### Node.js

```js
const https = require("https");
const fs = require("fs");

https.createServer({
  cert: fs.readFileSync("fullchain.pem"),
  key: fs.readFileSync("privkey.pem"),
}, (req, res) => res.end("ok")).listen(443);
```

### Snake AI game server

If you run the Snake AI server, set these in `.env`. The files are downloaded at startup:

```ini
HTTPS_CERT=url;https://github.com/OWNER/REPO/releases/download/cert/fullchain.pem
HTTPS_KEY=url;https://github.com/OWNER/REPO/releases/download/cert/privkey.pem
# or, from local files:
# HTTPS_CERT=file;./fullchain.pem
# HTTPS_KEY=file;./privkey.pem
# HTTPS_CERT=none   -> plain http
```

If `HTTPS_KEY` is left empty and `HTTPS_CERT` ends with `fullchain.pem`, the key is taken from `privkey.pem` in the same folder. Then open the server using the certificate hostname, for example `https://gateway-magic.huskydg.io.vn:8888`.

## Verify the files

```bash
# names, issuer and dates
openssl x509 -in fullchain.pem -noout -subject -issuer -dates -ext subjectAltName

# the key must match the certificate (both hashes must be identical)
openssl x509 -in fullchain.pem -noout -pubkey | openssl sha256
openssl pkey -in privkey.pem   -pubout          | openssl sha256

# validity check of a running server
openssl s_client -connect gateway-magic.huskydg.io.vn:443 -servername gateway-magic.huskydg.io.vn </dev/null 2>/dev/null | openssl x509 -noout -dates
```

## Keeping your copy fresh

A certificate is valid for a limited time (up to 90 days, see the release notes for the exact expiry date) and is renewed once a month. If someone reports the key as compromised, Let's Encrypt revokes the certificate immediately and a new one must be issued. So download the files again regularly and reload your server.

Example cron job (daily at 04:00) that only downloads when the file changed, then reloads nginx:

```bash
0 4 * * * cd /etc/ssl/gateway-magic && curl -fsSL -z fullchain.pem -o fullchain.pem https://github.com/OWNER/REPO/releases/download/cert/fullchain.pem && curl -fsSL -z privkey.pem -o privkey.pem https://github.com/OWNER/REPO/releases/download/cert/privkey.pem && nginx -s reload
```

## How it works

1. A scheduled GitHub Actions workflow (`.github/workflows/cert.yml`, 03:00 UTC on the 1st of every month, or run manually with *workflow_dispatch*) installs `certbot` with the Cloudflare DNS plugin.
2. Certbot requests one certificate for `gateway-magic.huskydg.io.vn` and `*.gateway-magic.huskydg.io.vn` using the **DNS-01** challenge. A temporary `_acme-challenge` TXT record is created and removed through the Cloudflare API, using the token stored in the repository secret `CF_TOKEN`.
3. The resulting `fullchain.pem`, `privkey.pem`, `cert.pem`, `chain.pem` and a zip are uploaded with `--clobber` to the GitHub Release tagged `cert`, so the download URLs stay stable.
4. The release notes are updated with the issue time and expiry date.

The Cloudflare token is never exposed: it lives only in GitHub Secrets and is not part of any published file.

## Run your own copy

If you want a certificate you control (strongly recommended for anything beyond testing), fork this repository and:

1. Make sure your domain's DNS zone is hosted on Cloudflare.
2. Create a Cloudflare API token with **Zone → DNS → Edit** and **Zone → Zone → Read** for that zone.
3. In your fork: *Settings → Secrets and variables → Actions → New repository secret*, name it `CF_TOKEN`.
4. Edit `DOMAIN` and `EMAIL` at the top of `.github/workflows/cert.yml`.
5. *Actions → Issue Let's Encrypt certificate → Run workflow*.
6. Keep the key **private** (private repository, or do not upload `privkey.pem` to a public release) if you do not want others to use your certificate.

Notes:

- GitHub disables scheduled workflows after 60 days without repository activity. If renewals stop, re-enable the workflow or run it manually.
- Let's Encrypt has rate limits (for example 5 certificates per week for the same set of names), so avoid running the workflow repeatedly.

## FAQ and troubleshooting

**Do I need permission to use it?**
No. It is intentionally open for testing.

**The browser still shows a warning.**
Check that you open the site by a covered hostname, not by an IP address or `localhost`. Check that the server serves `fullchain.pem` (not `cert.pem`), and that the system clock is correct.

**`unable to get local issuer certificate`.**
You used `cert.pem`. Switch to `fullchain.pem`.

**It worked yesterday and today the client says "revoked".**
The key is public, so someone can report it as compromised and Let's Encrypt will revoke the certificate. Download the new files once the workflow has issued a replacement (or run the workflow manually).

**Is `198.18.0.1` a real server on the internet?**
No. It is a reserved address that only means something inside the network of a device running Magic V2Ray (the one sharing the hotspot). Outside that network the name is not expected to work.

**Can I use it for production, logins or payments?**
No. Anyone holding the key can impersonate the hostname for any user who can be redirected to them.

**Why is `a.b.gateway-magic.huskydg.io.vn` not valid?**
A wildcard matches exactly one DNS label. Use `x.gateway-magic.huskydg.io.vn` instead.

**Can I get a certificate for my own name under this domain?**
Not at the moment. A per-user layout such as `*.username.gateway-magic.huskydg.io.vn`, where each user keeps their own private key, is planned.

## Disclaimer

This repository is provided "as is", without warranty of any kind. The certificate is shared for testing purposes only and may be revoked, replaced or removed at any time. You use it at your own risk. This project is not affiliated with or endorsed by Let's Encrypt or the Internet Security Research Group (ISRG).