# Dawarich via Tailscale

Deze configuratie laat **Dawarich** draaien met een publieke HTTPS-toegang via **Tailscale Funnel**, zonder dat je een domeinnaam of SSL-certificaat hoeft te beheren.

Publieke URL: `https://dawarich.yourdomainname.xxx`

---

## ⚙️ Installatie

1. Maak een `docker-compose.yml` met de volgende inhoud:

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
    command: ["bin/rails", "server", "-b", "127.0.0.1", "-p", "3000"]
    environment:
      RAILS_ENV: development
      REDIS_URL: redis://dawarich_redis:6379
      DATABASE_HOST: dawarich_db
      DATABASE_USERNAME: postgres
      DATABASE_PASSWORD: password
      DATABASE_NAME: dawarich_development
      APPLICATION_HOSTS: "localhost,dawarich.yourdomainname.xxx"
      TIME_ZONE: 
      APPLICATION_PROTOCOL: http
      SELF_HOSTED: "true"
      STORE_GEODATA: "true"
    depends_on:
      - dawarich_db
      - dawarich_redis
    restart: on-failure

  # 🔐 Tailscale met HTTPS Funnel
  tailscale:
    image: tailscale/tailscale:latest
    container_name: tailscale-dawarich
    environment:
      - TS_AUTHKEY= xxx
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

2. Vervang in de stack je **Tailscale-authkey** (`TS_AUTHKEY=`).
3. vervang uw domeinnaam door deze van tailscale
4. vul uw tijdzone in
5. Start alles met:
   ```bash
   docker compose up -d
   ```
6. Binnen enkele seconden is de app publiek bereikbaar via je Tailscale URL.

---

## 🧠 Belangrijk

- **Geen domeinnaam of SSL-certificaat nodig.**
- Funnel maakt automatisch een HTTPS-endpoint aan.
- Werkt enkel als de `tag:dawarich` nodeattribute in je ACL ‘funnel’ bevat.

---

## 🔧 Troubleshooting

**Fout:** `connection refused`  
➡ Controleer of de app luistert op `127.0.0.1:3000` binnen de container.

**Fout:** `Funnel not available; "funnel" node attribute not set`  
➡ Voeg `"attr": ["funnel"]` toe aan je `nodeAttrs` in de ACL.

---

Onderhouden door **Kevin Thijs**
