# Dawarich via Tailscale

Deze configuratie laat **Dawarich** draaien met een publieke HTTPS-toegang via **Tailscale Funnel**, zonder dat je een domeinnaam of SSL-certificaat hoeft te beheren.

Publieke URL: `https://dawarich.yourdomainname.xxx`

---

## ⚙️ Installatie

1. Maak een `docker-compose.yml` met de volgende inhoud:

```yaml
version: "3.9"
# (volledige stack zoals gebruikt voor Dawarich)
```

2. Vervang in de stack je **Tailscale-authkey** (`TS_AUTHKEY=`).  
3. Start alles met:
   ```bash
   docker compose up -d
   ```
4. Binnen enkele seconden is de app publiek bereikbaar via je Tailscale URL.

---

## 🧠 Belangrijk

- **Geen domeinnaam of SSL-certificaat nodig.**
- Funnel maakt automatisch een HTTPS-endpoint aan.
- Werkt enkel als de `tag:dawarich` nodeattribute in je ACL ‘funnel’ bevat.

---

## 🔧 Troubleshooting

**Fout:** `connection refused`  
➡ Controleer of de app luistert op `127.0.0.1:3001` binnen de container.

**Fout:** `Funnel not available; "funnel" node attribute not set`  
➡ Voeg `"attr": ["funnel"]` toe aan je `nodeAttrs` in de ACL.

---

Onderhouden door **Kevin Thijs**
