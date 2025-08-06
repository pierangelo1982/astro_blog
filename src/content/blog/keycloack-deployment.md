---
title: 'Deploying Keycloak: A Step-by-Step Guide for Production'
description: 'A comprehensive guide to installing and deploying Keycloak with PostgreSQL and NGINX.'
pubDate: 'Aug 06 2025'
heroImage: '../../assets/blog-placeholder-1.jpg'
---

This document details how to install and deploy Keycloak in both development and production environments. The setup integrates PostgreSQL as the database and uses an NGINX reverse proxy for HTTPS access. This guide is intended for Debian and Ubuntu-based systems.

## Table of Contents
- [Prerequisites](#prerequisites)
- [Development Mode](#development-mode)
- [Production Mode](#production-mode)
  - [Generating Certificates](#1-generating-certificates)
  - [Creating a Keystore](#2-creating-a-keystore-with-keytool)
- [Docker and PostgreSQL Configuration](#docker-and-postgresql-configuration)
  - [Create a Docker Network](#1-create-a-docker-network)
  - [Set Up PostgreSQL](#2-set-up-postgresql)
- [Building a Custom Docker Image for Keycloak](#building-a-custom-docker-image-for-keycloak)
  - [Prepare the Directory](#1-prepare-the-directory)
  - [Dockerfile Contents](#2-dockerfile-contents)
  - [Build the Docker Image](#3-build-the-docker-image)
  - [Run the Container](#4-run-the-container)
- [Configuring NGINX as a Reverse Proxy](#configuring-nginx-as-a-reverse-proxy)
  - [Create NGINX Site Configuration](#1-create-the-nginx-site-configuration-file)
  - [Configure NGINX](#2-configure-the-file-with-the-following-content)
  - [Enable the Site and Restart NGINX](#3-enable-the-site-and-restart-nginx)
- [Accessing Keycloak](#accessing-keycloak)
- [Final Notes](#final-notes)

---

## Prerequisites

- **Java:** Ensure that OpenJDK is installed. For example, on Debian/Ubuntu you can install the default JDK using:
  ```bash
  sudo apt update
  sudo apt install default-jdk
  ```
- **Docker:** Docker must be installed to run the Keycloak and PostgreSQL containers.
- **Certbot (for NGINX):** Required for managing SSL certificates with Let's Encrypt.

---

## Development Mode

To quickly test Keycloak in development mode, you can use the following Docker command.

```bash
docker run -p 8090:8080 \
  -e KEYCLOAK_ADMIN=admin \
  -e KEYCLOAK_ADMIN_PASSWORD=<ADMIN_PASSWORD> \
  --dns=8.8.8.8 \
  -v keycloak_data:/opt/keycloak/data \
  quay.io/keycloak/keycloak:23.0.4 \
  start-dev 2>&1 > /dev/null &
```

**Explanation:**
- **Port Mapping:** The container exposes port `8080`, which is mapped to port `8090` on the host.
- **Admin Credentials:** Environment variables set the admin username and password.
- **DNS:** Uses Google's DNS (`8.8.8.8`) for name resolution.
- **Volume:** Data is persisted in a Docker volume named `keycloak_data`.
- **Version:** This command uses Keycloak version `23.0.4` in development mode (`start-dev`).

---

## Production Mode

For a production deployment, you must configure HTTPS certificates and properly set up the container.

### 1. Generating Certificates

Generate the necessary certificates using OpenSSL:

- **Generate the Private Key:**
  ```bash
  openssl genrsa -out keycloak.key 2048
  ```
- **Create a Self-Signed Certificate (valid for 365 days):**
  ```bash
  openssl req -new -x509 -sha256 -key keycloak.key -out keycloak.crt -days 365
  ```
- **Create the PEM File:**
  Rename the private key file and concatenate the certificate:
  ```bash
  mv keycloak.key keycloak.pem
  cat keycloak.crt >> keycloak.pem
  ```
  
*> Note: You may need to adjust parameters (like CN, O, C, etc.) during certificate generation.*

### 2. Creating a Keystore with keytool

Use the `keytool` command to generate a key pair and create the keystore:

```bash
sudo keytool -genkeypair -alias localhost \
  -keyalg RSA \
  -keysize 2048 \
  -validity 365 \
  -keystore server.keystore \
  -dname "CN=Server Administrator,O=OrgName,C=IN" \
  -keypass <KEY_PASSWORD> \
  -storepass <KEY_PASSWORD>
```

After completion, you will have three files:
- `keycloak.crt`
- `keycloak.pem`
- `server.keystore` (which should be converted to PKCS12 format and then renamed)

These files will be provided as arguments when launching Keycloak:
- `--https-certificate-file=<path>/keycloak.crt`
- `--https-certificate-key-file=<path>/keycloak.pem`
- `--https-trust-store-file=<path>/server.keystore`

---

## Docker and PostgreSQL Configuration

### 1. Create a Docker Network

Create a dedicated Docker network for container communication:

```bash
docker network create kcnetwork
```

### 2. Set Up PostgreSQL

Create a Docker volume for PostgreSQL data persistence:

```bash
docker volume create pgdata
```

Run the PostgreSQL container:

```bash
docker run --name kc_pg_cont \
  --network kcnetwork \
  -e POSTGRES_PASSWORD=<POSTGRES_PASSWORD> \
  -p 5432:5432 \
  --restart="always" \
  -v pgdata:/var/lib/postgresql/data \
  -d postgres
```

---

## Building a Custom Docker Image for Keycloak

### 1. Prepare the Directory

Create a directory for Keycloak files:

```bash
mkdir keycloak
```

Inside this directory, create a `Dockerfile`:

```bash
touch keycloak/Dockerfile
```

Copy the certificate and keystore files into this directory:

```bash
cp keycloak.crt keycloak/
cp keycloak.pem keycloak/
cp server.keystore keycloak/server.keystore.p12
```

### 2. Dockerfile Contents

Place the following content in your `Dockerfile`. **Note:** Passwords have been anonymized for security.

```dockerfile
# Use the latest Keycloak image
FROM quay.io/keycloak/keycloak:latest

USER root

# Copy the certificates and keystore into the configuration directory
COPY keycloak.crt /opt/keycloak/conf/keycloak.crt
COPY keycloak.pem /opt/keycloak/conf/keycloak.pem
COPY server.keystore.p12 /opt/keycloak/conf/server.keystore.p12

# Debug: list the copied files to confirm their presence
RUN ls -la /opt/keycloak/conf/

# Set the trust store type to PKCS12
ENV KC_HTTPS_TRUST_STORE_TYPE=PKCS12

# Database configuration
ENV KC_DB=postgres
ENV KC_DB_USERNAME=postgres
ENV KC_DB_PASSWORD=<POSTGRES_PASSWORD>
ENV KC_DB_URL=jdbc:postgresql://kc_pg_cont:5432/postgres

# Keycloak admin credentials
ENV KEYCLOAK_ADMIN=admin
ENV KEYCLOAK_ADMIN_PASSWORD=<ADMIN_PASSWORD>

# (Optional) Set correct permissions on the files
RUN chmod 600 /opt/keycloak/conf/keycloak.pem
RUN chmod 600 /opt/keycloak/conf/keycloak.crt
RUN chmod 600 /opt/keycloak/conf/server.keystore.p12

# Start the Keycloak server with HTTPS options and specified hostname
ENTRYPOINT ["/opt/keycloak/bin/kc.sh", "start", \
  "--https-port=9443", "--http-port=8095", \
  "--https-certificate-file", "/opt/keycloak/conf/keycloak.crt", \
  "--https-certificate-key-file", "/opt/keycloak/conf/keycloak.pem", \
  "--https-trust-store-file", "/opt/keycloak/conf/server.keystore.p12", \
  "--https-trust-store-password", "<KEYSTORE_PASSWORD>", \
  "--hostname", "keycloak-develop.example.com", \
  "--https-trust-store-type", "PKCS12"]
```

### 3. Build the Docker Image

Build your custom Keycloak image:

```bash
docker build -t kcprod_v1 .
```

### 4. Run the Container

Start the Keycloak container on the created network:

```bash
docker run --name kctest_container --network kcnetwork -p 9443:9443 -p 8095:8095 -d kcprod_v1
```

---

## Configuring NGINX as a Reverse Proxy

To expose Keycloak under your domain and manage HTTPS, set up NGINX.

### 1. Create the NGINX Site Configuration File

For example, create a file at `/etc/nginx/sites-available/keycloak-develop.example.com`:
```bash
sudo touch /etc/nginx/sites-available/keycloak-develop.example.com
```

### 2. Configure the File with the Following Content

```nginx
server {
    server_name keycloak-develop.example.com;
    client_max_body_size 100M;

    location / {
        proxy_pass https://keycloak-develop.example.com:9443;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Disable SSL verification for Keycloak if necessary:
        proxy_ssl_verify off;
    }

    listen 443 ssl; # Managed by Certbot
    ssl_certificate /etc/letsencrypt/live/keycloak-develop.example.com/fullchain.pem; # Managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/keycloak-develop.example.com/privkey.pem; # Managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # Managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # Managed by Certbot
}

server {
    if ($host = keycloak-develop.example.com) {
        return 301 https://$host$request_uri;
    }
    
    server_name keycloak-develop.example.com;
    
    listen 80;
    return 404;
}
```

### 3. Enable the Site and Restart NGINX

Link the configuration file:

```bash
sudo ln -s /etc/nginx/sites-available/keycloak-develop.example.com /etc/nginx/sites-enabled/
```

Test the configuration:

```bash
sudo nginx -t
```

Restart NGINX:

```bash
sudo systemctl restart nginx
```

---

## Accessing Keycloak

After the configuration, Keycloak will be available at:

`https://keycloak-develop.example.com/`

Ensure that DNS records point to your server and that Let's Encrypt certificates are properly obtained and installed.

---

## Final Notes

- **Security:** Verify the permissions on the certificate and keystore files. Consider using secure environment variables for passwords.
- **Persistence:** Make sure Docker volumes for both Keycloak and PostgreSQL are correctly configured to persist data.
- **Monitoring and Logging:** Set up monitoring to observe container performance in production.
- **Updates:** Plan for periodic updates to Keycloak, the operating system, and security components.
- **Placeholders:** Remember to replace placeholder values (e.g., `<ADMIN_PASSWORD>`, `<POSTGRES_PASSWORD>`, `<KEYSTORE_PASSWORD>`, and domain names) with your actual production values.
---