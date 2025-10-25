# Dawarich via Tailscale

This setup runs **Dawarich** with public HTTPS access using **Tailscale Funnel**, without requiring a domain name or SSL certificate.

Public URL: `https://dawarich.yourdomainname.xxx`

---

## ⚙️ Setup

1. Create a `docker-compose.yml` file with the following content:

```yaml
version: "3.9"
# (full Dawarich stack as used in production)
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
