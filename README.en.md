# Dawarich via Tailscale

This setup runs **Dawarich** with public HTTPS access using **Tailscale Funnel**, without requiring a domain name or SSL certificate.

Public URL: `https://dawarich.yourdomainname.xxx`

---

## ⚙️ Setup

1. Create a `docker-compose.yml` file with the following content:

```yaml
version: "3.9"

volumes:
  dawarich_db_data:
  dawarich_shared:
  dawarich_public:
  dawarich_watched:
  dawarich_storage:
  tailscale_state_dawarich:

networks:
  dawarich:
    driver: bridge

services:
  # 🧠 Redis Cache
  dawarich_redis:
    image: redis:7.4-alpine
    container_name: dawarich_redis
    command: redis-server
    networks:
      - dawarich
    volumes:
      - dawarich_shared:/data
    restart: always

  # 🗄️ PostgreSQL + PostGIS database
  dawarich_db:
    image: postgis/postgis:14-3.5-alpine
    container_name: dawarich_db
    networks:
      - dawarich
    volumes:
      - dawarich_db_data:/var/lib/postgresql/data
      - dawarich_shared:/var/shared
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    restart: always

  # 🌍 Dawarich Web App (Rails)
  dawarich_app:
    image: freikin/dawarich:latest
    container_name: dawarich_app
    networks:
      - dawarich
    volumes:
      - dawarich_public:/var/app/public
      - dawarich_watched:/var/app/tmp/imports/watched
      - dawarich_storage:/var/app/storage
      - dawarich_db_data:/dawarich_db_data
    command: ["bin/rails", "server", "-b", "127.0.0.1", "-p", "3001"]
    environment:
      RAILS_ENV: development
      REDIS_URL: redis://dawarich_redis:6379
      DATABASE_HOST: dawarich_db
      DATABASE_USERNAME: postgres
      DATABASE_PASSWORD: password
      DATABASE_NAME: dawarich_development
      APPLICATION_HOSTS: "localhost,https://dawarich.yourdomainname.xxx"
      TIME_ZONE: Europe/Brussels
      APPLICATION_PROTOCOL: http
      SELF_HOSTED: "true"
      STORE_GEODATA: "true"
      LOCATIONIQ_API_KEY: "pk.c01641f72c20dc7aa0826acb7c35761b"
    depends_on:
      - dawarich_db
      - dawarich_redis
    restart: on-failure

  # 🔐 Tailscale met HTTPS Funnel
  tailscale:
    image: tailscale/tailscale:latest
    container_name: tailscale-dawarich
    environment:
      - TS_AUTHKEY=tskey-auth-REPLACE-ME
      - TS_STATE_DIR=/var/lib/tailscale
      - TS_HOSTNAME=dawarich
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    volumes:
      - /dev/net/tun:/dev/net/tun
      - tailscale_state_dawarich:/var/lib/tailscale
    networks:
      - dawarich
    command: >
      sh -c "
        echo '🌀 Starting tailscaled...' &&
        tailscaled --state=/var/lib/tailscale/tailscaled.state &
        echo '⏳ Waiting for tailscaled...' &&
        until tailscale status >/dev/null 2>&1; do sleep 1; done &&
        echo '🔑 Logging in...' &&
        tailscale up --authkey=${TS_AUTHKEY} --hostname=dawarich --accept-routes --accept-dns=false --advertise-tags=tag:dawarich &&
        echo '🌍 Enabling public HTTPS funnel...' &&
        tailscale funnel 127.0.0.1:3001 --bg &&
        echo '✅ Funnel active. Ready on https://dawarich.yourdomainname.xxx' &&
        tail -f /dev/null
      "
    restart: always

```

2. Replace your **Tailscale auth key** (`TS_AUTHKEY=`).  
3. Launch the stack with:
   ```bash
   docker compose up -d
   ```
4. Within seconds, the app will be publicly available through your Tailscale URL.

---

## 🧠 Notes

- **No domain name or SSL certificate needed.**
- Funnel automatically creates a secure HTTPS endpoint.
- Works only if your ACL grants `"attr": ["funnel"]` to the `tag:dawarich` node.

---

## 🔧 Troubleshooting

**Error:** `connection refused`  
➡ Check if the app is listening on `127.0.0.1:3001` inside the container.

**Error:** `Funnel not available; "funnel" node attribute not set`  
➡ Add `"attr": ["funnel"]` to your `nodeAttrs` in the ACL.

---

Maintained by **Kevin Thijs**
