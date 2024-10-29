---
title: NGINX Proxy Manager
image: 
  path: https://i.ibb.co/fvhfFMv/10416564-m.jpg
sitemap: true
categories: [Docker]
tag: [HowTo]
comments: false
---

## Installation

```bash
sudo mkdir /opt/nginx
cd /opt/nginx
sudo nano compose.yaml
```

## Compose.yaml erstellen

```yaml
services:
  app:
    image: jc21/nginx-proxy-manager:latest
    restart: unless-stopped
    container_name: nginx-proxy-manager
    ports:
      # These ports are in format <host-port>:<container-port>
      - 80:80 # Public HTTP Port
      - 443:443 # Public HTTPS Port
      - 81:81 # Admin Web Port 
      # Add any other Stream port you want to expose
      # - '21:21' # FTP
      # Uncomment this if IPv6 is not enabled on your host
      # DISABLE_IPV6: 'true'
    volumes:
      - /opt/nginx/data:/data
      - /opt/nginx/letsencrypt:/etc/letsencrypt
networks:
  default:
    name: reverse-proxy
```

```bash
sudo docker compose up -d
```

Got to `http://ip-adress:81`
User: `admin@example.com`
Password: `changeme`
