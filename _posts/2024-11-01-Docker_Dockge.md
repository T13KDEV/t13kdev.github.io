---
title: DOCKGE
image: 
  path: https://i.ibb.co/fvhfFMv/10416564-m.jpg
sitemap: true
categories: [Docker]
tag: [HowTo]
comments: false
---

## Installation

```bash
mkdir -p /opt/stacks /opt/dockge
cd /opt/dockge
curl https://raw.githubusercontent.com/louislam/dockge/master/compose.yaml --output compose.yaml
sudo nano compose.yaml
```

## Compose.yaml erstellen

```yaml
services:
  dockge:
    image: louislam/dockge:1
    container_name: docker-dockge
    restart: unless-stopped
    #ports: -> over reverse-network and nginx
      # Host Port : Container Port
      #- 5001:5001 #done over reverse-network with nginx
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./data:/app/data

      # If you want to use private registries, you need to share the auth file with Dockge:
      # - /root/.docker/:/root/.docker

      # Stacks Directory
      # ⚠️ READ IT CAREFULLY. If you did it wrong, your data could end up writing into a WRONG PATH.
      # ⚠️ 1. FULL path only. No relative path (MUST)
      # ⚠️ 2. Left Stacks Path === Right Stacks Path (MUST)
      - /opt/stacks:/opt/stacks
    environment:
      # Tell Dockge where is your stacks directory
      - DOCKGE_STACKS_DIR=/opt/stacks
    networks:
      - reverse-proxy 
networks:
  reverse-proxy:
    external: true
```

```bash
sudo docker compose up -d
```


