# Maintenance Page Server

A flexible HAProxy-based maintenance page server that serves custom maintenance pages for multiple domains. This solution uses Docker Compose for easy deployment and supports SSL/TLS certificates.

## Table of contents

- [Maintenance Page Server](#maintenance-page-server)
  - [Table of contents](#table-of-contents)
  - [Overview](#overview)
  - [Quick Start](#quick-start)
    - [1. Start the Server](#1-start-the-server)
    - [2. Check Status](#2-check-status)
    - [3. View Logs](#3-view-logs)
    - [4. Stop the Server](#4-stop-the-server)
  - [Adding Custom Maintenance Pages](#adding-custom-maintenance-pages)
    - [Step 1: Create the HTML Page](#step-1-create-the-html-page)
    - [Step 2: Configure HAProxy](#step-2-configure-haproxy)
    - [Step 3: Restart HAProxy](#step-3-restart-haproxy)
  - [Testing](#testing)
    - [Test HTTP](#test-http)
    - [Test HTTPS](#test-https)
  - [Directory Structure](#directory-structure)
  - [SSL/TLS Certificates](#ssltls-certificates)
    - [Certificate File Format](#certificate-file-format)
    - [Issuing Certificates](#issuing-certificates)
      - [Option 1: Single Domain Certificate (HARICA ACME)](#option-1-single-domain-certificate-harica-acme)
      - [Option 2: Wildcard Domain Certificate (HARICA ACME)](#option-2-wildcard-domain-certificate-harica-acme)
      - [Option 3: Let's Encrypt (HTTP-01 Challenge)](#option-3-lets-encrypt-http-01-challenge)


## Overview

This project provides a lightweight, configurable maintenance page server with:
- **Multi-domain support**: Serve different maintenance pages for different domains
- **SSL/TLS support**: Automatic SSL certificate handling
- **Docker-based**: Easy deployment with Docker Compose
- **HAProxy**: High-performance load balancer in maintenance mode

By default, it handles domains ending with `nau.edu.pt`, `educast.fccn.pt`, or `fccn.pt`.

## Quick Start

### 1. Start the Server

```bash
docker compose up -d
```

### 2. Check Status

```bash
docker compose ps
```

### 3. View Logs

```bash
# View all logs
docker compose logs

# Follow logs (tail 100 lines)
docker compose logs -f --tail 100
```

### 4. Stop the Server

```bash
docker compose stop
```

## Adding Custom Maintenance Pages

### Step 1: Create the HTML Page

Add a new HTML file to the `pages/` folder:

```bash
cp pages/nau.html pages/a-service.html
# Edit pages/a-service.html with your content
```

### Step 2: Configure HAProxy

Edit `haproxy/haproxy.cfg` and add routing rules for your domain:

```haproxy
acl a_service_domain_acl hdr_end(host) -i a-service.fccn.pt
http-request return status 503 content-type text/html file /pages/a-service.html if a_service_domain_acl
```

**Rule components:**
- `acl a_service_domain_acl`: Define an Access Control List (ACL) named `a_service_domain_acl`
- `hdr_end(host) -i a-service.fccn.pt`: Match requests where the Host header ends with `a-service.fccn.pt` (case-insensitive)
- `http-request return status 503`: Return HTTP 503 (Service Unavailable) status
- `content-type text/html`: Set the Content-Type header
- `file /pages/a-service.html`: Serve the specified HTML file
- `if a_service_domain_acl`: Apply this rule only when the ACL matches

### Step 3: Restart HAProxy

```bash
docker compose restart
```

> **Note:** If HAProxy doesn't load the new configuration immediately, try restarting again or use `docker compose down && docker compose up -d`.

## Testing

You can test the maintenance pages locally using `curl` with custom host resolution:

### Test HTTP

```bash
curl -v --resolve educast.fccn.pt:80:127.0.0.1 http://educast.fccn.pt/
```

### Test HTTPS

```bash
curl -v --resolve www.nau.edu.pt:443:127.0.0.1 https://www.nau.edu.pt/cursos
```

The `--resolve` flag maps the domain to localhost without modifying `/etc/hosts`.

## Directory Structure

```
.
├── docker-compose.yml       # Docker Compose configuration
├── haproxy/
│   └── haproxy.cfg         # HAProxy routing rules
├── pages/                  # HTML maintenance pages
│   ├── educast.html
│   ├── fccn.html
│   └── nau.html
└── certs/                  # SSL/TLS certificates
```

## SSL/TLS Certificates

### Certificate File Format

Place certificates in the `certs/` folder with the following naming convention:
- **Certificate**: `domain_name.crt` (full chain)
- **Private Key**: `domain_name.crt.key`

Example:
```
certs/
├── www_nau_edu_pt.crt
├── www_nau_edu_pt.crt.key
├── wildcard_nau_edu_pt.crt
└── wildcard_nau_edu_pt.crt.key
```

### Issuing Certificates

Next there are a couple of examples on how to issue certificates using certbot on a docker container.

Note: The HAProxy won't reload automatically the new certificates on the `certs` folder.
It is possible, but it is too complicated for this kind of service.
So a "normal" stop start would do the trick.

#### Option 1: Single Domain Certificate (HARICA ACME)

Use this method for specific domains with HARICA or other ACME providers.

**Example:** `a-service.fccn.pt`

```bash
# Create a temporary folder for certificates
mkdir certbot

# Issue certificate via HARICA ACME
docker run --rm \
  --volume ./certbot:/etc/letsencrypt/ \
  --cpus 1 \
  --memory 100M \
  --name certbot \
  certbot/certbot:latest \
  certonly \
  --standalone \
  --non-interactive \
  --agree-tos \
  --server https://acme-v02.harica.gr/acme/XXXXXXX-YYYY-ZZZZ-WWW-ZZZZZZZZZZ/directory \
  --eab-kid AAAAAAAAAAAAAAA \
  --eab-hmac-key BBBBBBBBBBBBBBBBBBBBBBB \
  --cert-name a-service.fccn.pt \
  --email someone@fccn.pt \
  -d a-service.fccn.pt

# Change file ownership
sudo chown -R $(id -u):$(id -g) certbot

# Copy certificate and key to certs folder
cp certbot/live/a-service.fccn.pt/fullchain.pem certs/a-service_fccn_pt.crt
cp certbot/live/a-service.fccn.pt/privkey.pem certs/a-service_fccn_pt.crt.key

# Clean up temporary folder
sudo rm -rf certbot
```

**Parameters:**
- `--server`: ACME server URL (replace with your ACME provider)
- `--eab-kid` and `--eab-hmac-key`: External Account Binding credentials from your CA
- `--email`: Contact email for certificate notifications
- `-d`: Domain name for the certificate

#### Option 2: Wildcard Domain Certificate (HARICA ACME)

Use wildcard certificates to cover all subdomains.

**Example:** `*.nau.edu.pt`

> **⚠️ Important:** For maintenance services with wildcard certificates, avoid enabling HTTP/2 on HAProxy to prevent "HTTP/2 connection coalescing" issues where browsers may reuse connections unexpectedly.

```bash
# Create a temporary folder for certificates
mkdir certbot

# Issue wildcard certificate
docker run --rm \
  --volume ./certbot:/etc/letsencrypt/ \
  --cpus 1 \
  --memory 100M \
  --name certbot \
  certbot/certbot:latest \
  certonly \
  --standalone \
  --non-interactive \
  --agree-tos \
  --server https://acme-v02.harica.gr/acme/XXXXXXX-YYYY-ZZZZ-WWW-ZZZZZZZZZZ/directory \
  --eab-kid AAAAAAAAAAAAAAA \
  --eab-hmac-key BBBBBBBBBBBBBBBBBBBBBBB \
  --cert-name wildcard.nau.edu.pt \
  --email someone@fccn.pt \
  -d "*.nau.edu.pt"

# Change file ownership
sudo chown -R $(id -u):$(id -g) certbot

# Copy certificate and key to certs folder
cp certbot/live/wildcard.nau.edu.pt/fullchain.pem certs/wildcard_nau_edu_pt.crt
cp certbot/live/wildcard.nau.edu.pt/privkey.pem certs/wildcard_nau_edu_pt.crt.key

# Clean up temporary folder
sudo rm -rf certbot
```

#### Option 3: Let's Encrypt (HTTP-01 Challenge)

Use Let's Encrypt for free SSL certificates via HTTP-01 challenge.

**How it works:**
1. Let's Encrypt requests a challenge file at `/.well-known/acme-challenge/TOKEN`
2. HAProxy proxies these requests to Certbot on port 8888 (configured in `haproxy.cfg`)
3. Certbot responds with the correct challenge response
4. Let's Encrypt verifies and issues the certificate

**Requirements:**
- The domain must be publicly accessible
- HAProxy must be running and configured to proxy ACME challenges

**Commands:**

```bash
# Create a temporary folder for certificates
mkdir certbot

# Issue certificate via Let's Encrypt
docker run --rm \
  --volume ./certbot:/etc/letsencrypt/ \
  --cpus 1 \
  --memory 100M \
  --name certbot \
  certbot/certbot:latest \
  certonly \
  --standalone \
  --non-interactive \
  --agree-tos \
  --http-01-port=8888 \
  --cert-name b-service.fccn.pt \
  --email someone@fccn.pt \
  -d b-service.fccn.pt

# Change file ownership
sudo chown -R $(id -u):$(id -g) certbot

# Copy certificate and key to certs folder
cp certbot/live/b-service.fccn.pt/fullchain.pem certs/b-service_fccn_pt.crt
cp certbot/live/b-service.fccn.pt/privkey.pem certs/b-service_fccn_pt.crt.key

# Clean up temporary folder
sudo rm -rf certbot
```
