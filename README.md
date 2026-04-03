# VLESS + REALITY Proxy Setup Guide

This guide provides step-by-step instructions for deploying a stealthy proxy server using Xray-core (via the 3x-ui panel) with the VLESS + REALITY protocol. This setup is specifically designed to bypass advanced Deep Packet Inspection (DPI) and active probing, such as China's Great Firewall (GFW), without interfering with other services like Minecraft.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [How to Launch the Service](#how-to-launch-the-service)
3. [How to Create a VLESS + REALITY Inbound](#how-to-create-a-vless--reality-inbound)
4. [How to Create New Clients (for multiple devices)](#how-to-create-new-clients)
5. [Setting up the Happ App (Client Side)](#setting-up-the-happ-app-client-side)
6. [Troubleshooting](#troubleshooting)
7. [Advanced: Adding a Traefik Reverse Proxy](#advanced-adding-a-traefik-reverse-proxy)

---

## Prerequisites

- A Linux server (Ubuntu/Debian recommended).
- **Port 443 must be completely free.** Ensure older proxies like Shadowsocks/v2ray-plugin or Nginx are stopped.
- Docker and Docker Compose installed on your server.
- System time strictly synchronized (NTP).

---

## How to Launch the Service

We use Docker Compose to keep the installation clean and isolated from other services (like Minecraft servers).

1. Create a directory for the panel and navigate into it:

    ```bash
    mkdir 3x-ui-panel && cd 3x-ui-panel
    ```

2. Create a `docker-compose.yml` file:

    ```yaml
    services:
      3x-ui:
        image: ghcr.io/mhsanaei/3x-ui:latest
        container_name: 3x-ui
        restart: unless-stopped
        ports:
          - "2053:2053"  # Management Panel Port
          - "443:443"    # VLESS + REALITY Proxy Port
          - "80:80"      # Optional HTTP port
        volumes:
          - ./3x-ui-db:/etc/x-ui/
        environment:
          - TZ=Asia/Shanghai
    ```

3. Start the container in the background:

    ```bash
    docker compose up -d
    ```

4. Access the web dashboard by navigating to `http://<your-server-ip>:2053` in your browser.
    - Default Username: `admin`
    - Default Password: `admin`
    - **Important:** Go to **Panel Settings** immediately after logging in and change these default credentials.

---

## How to Create a VLESS + REALITY Inbound

This configuration will mask your proxy traffic so it appears as standard HTTPS traffic to a legitimate, unblocked website (e.g., Microsoft).

1. In the 3x-ui panel, navigate to the **Inbounds** tab and click **Add Inbound**.
2. Configure the following parameters:

    **Basic Settings**
    | Field | Value |
    |---|---|
    | Remark | `VLESS-REALITY` (or any name you prefer) |
    | Protocol | `vless` |
    | Listening IP | *(leave blank)* |
    | Port | `443` |

    **Client Settings**
    | Field | Value |
    |---|---|
    | Email | `device-1` (or your device's name) |
    | Flow | `xtls-rprx-vision` *(crucial for bypassing DPI)* |

    **Sniffing Settings**
    | Field | Value |
    |---|---|
    | Sniffing | `ON` |
    | Route Only | `ON` |

    **Transport Settings**
    | Field | Value |
    |---|---|
    | Network | `tcp` |
    | Security | `reality` |

    **REALITY Settings (Camouflage)**
    | Field | Value |
    |---|---|
    | uTLS | `chrome` or `firefox` |
    | Dest | `www.microsoft.com:443` (or `itunes.apple.com:443`) |
    | Server Names | `www.microsoft.com` *(must match Dest, without port — multiple values allowed: `www.microsoft.com`, `microsoft.com`)* |
    | Private/Public Key | Click **"Get new keys"** to auto-generate |
    | ShortIds | Click **"Generate one"** |

3. Click **Create** to save the inbound.
4. Click the **QR Code** or **Clipboard** icon next to the created inbound to copy your connection link (starts with `vless://`).

---

## How to Create New Clients

If you want to connect a laptop or a friend's device, do **not** create a new inbound on port 443. Instead, add a new client (user) to your existing inbound.

1. Go to the **Inbounds** tab in 3x-ui.
2. Find your existing `VLESS-REALITY` connection and click the **Edit** (pencil) icon.
3. Scroll down to the **Clients** section.
4. Click the **+** button to add a new user.
5. Give the new client a distinct email/name (e.g., `laptop`).
6. Make sure **Flow** is set to `xtls-rprx-vision`.
7. Click **Save**.
8. Expand the inbound details on the main page to find the specific connection link (QR code / clipboard) for this new client.

---

## Setting up the Happ App (Client Side)

Happ natively supports Xray-core and VLESS + REALITY.

### Step 1: Import the Server

1. Copy the `vless://` link from the 3x-ui panel to your clipboard.
2. Open the Happ app.
3. Tap the **Add (+)** button in the top right corner.
4. Select **Import from Clipboard** (or scan the QR code if using a separate screen). Happ will auto-fill all REALITY parameters.

### Step 2: Optimal Client Settings

> ⚠️ Do **NOT** enable **Fragmentation**. Normal browsers do not fragment connections to Microsoft. Enabling this strips away the REALITY camouflage and makes your traffic highly suspicious to the GFW.

Rely entirely on the `xtls-rprx-vision` flow to manage your packets.

### Step 3: Connect

1. Select the newly imported server to make it the active profile.
2. Tap the main **Connect** button.
3. *(Optional)* Run the latency test inside the app to verify connectivity.

---

## Troubleshooting

**App shows Timeout / `-1ms` Ping**

- **System Time:** VLESS is highly sensitive to time drift. Ensure the clock on your client device (especially laptops) is synced via internet time and is within 90 seconds of the server's time.
- **Firewall:** Windows Defender or third-party antivirus software often blocks proxy apps. Add an exclusion for Happ.

**Server IP cannot be pinged via Command Line**

This is completely normal. REALITY drops ICMP (ping) requests to maintain stealth. Only test connectivity through the proxy client itself.

---

## Advanced: Adding a Traefik Reverse Proxy

If you want to secure your 3x-ui web panel behind a custom URL (e.g., `panel.yourdomain.com`) with automated Let's Encrypt SSL certificates, you can use Traefik.

> ⚠️ **CRITICAL WARNING:** You cannot simply put Traefik in front of Xray — it will intercept the TLS connection and destroy your REALITY camouflage. Instead, we use **SNI Multiplexing**. Traefik inspects incoming traffic and routes it:
> - Requests for your **fake domain** (e.g., `www.microsoft.com`) → raw encrypted traffic passed directly to Xray.
> - Requests for your **real domain** (e.g., `panel.yourdomain.com`) → Traefik handles SSL and routes to the 3x-ui panel.

### Step 1: Update Panel Settings First

Before changing your Docker configuration, log into the 3x-ui panel, edit your VLESS-REALITY inbound, and change the port from `443` to `4433`. Xray will now listen internally on `4433`, while Traefik will listen externally on `443`.

> **Note:** Update your Happ client to keep port `443` manually, as Traefik handles the external port.

### Step 2: The New `docker-compose.yml`

Replace your existing `docker-compose.yml` with this advanced setup:

```yaml
services:
  3x-ui:
    image: ghcr.io/mhsanaei/3x-ui:latest
    container_name: 3x-ui
    restart: unless-stopped
    volumes:
      - ./3x-ui-db:/etc/x-ui/
    environment:
      - TZ=Asia/Shanghai
    networks:
      - proxy-net
    labels:
      - "traefik.enable=true"

      # 1. TCP Router: Xray REALITY Stealth Passthrough
      - "traefik.tcp.routers.xray-reality.rule=HostSNI(`www.microsoft.com`)" # Must match your REALITY Dest
      - "traefik.tcp.routers.xray-reality.entrypoints=websecure"
      - "traefik.tcp.routers.xray-reality.tls.passthrough=true"
      - "traefik.tcp.services.xray-reality.loadbalancer.server.port=4433" # Matches internal Xray port

      # 2. HTTP Router: 3x-ui Web Panel via Custom URL
      - "traefik.http.routers.xui-panel.rule=Host(`panel.yourdomain.com`)" # Change to your actual domain
      - "traefik.http.routers.xui-panel.entrypoints=websecure"
      - "traefik.http.routers.xui-panel.tls=true"
      - "traefik.http.routers.xui-panel.tls.certresolver=cloudflare"
      - "traefik.http.services.xui-panel.loadbalancer.server.port=2053"

  traefik:
    image: traefik:v3.6  # Use v3.6+ to avoid Docker API version errors
    container_name: traefik
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    environment:
      # Generate via: Cloudflare Dashboard → API Tokens → "Edit zone DNS" template
      - CF_DNS_API_TOKEN=your_cloudflare_api_token_here
    command:
      - "--api.insecure=false"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--providers.file.filename=/etc/traefik/dynamic.yml"
      - "--providers.file.watch=true"
      - "--certificatesresolvers.cloudflare.acme.dnschallenge=true"
      - "--certificatesresolvers.cloudflare.acme.dnschallenge.provider=cloudflare"
      - "--certificatesresolvers.cloudflare.acme.email=YOUR_EMAIL@example.com" # Change this
      - "--certificatesresolvers.cloudflare.acme.storage=/letsencrypt/acme.json"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./letsencrypt:/letsencrypt
      - ./dynamic.yml:/etc/traefik/dynamic.yml:ro
    networks:
      - proxy-net

networks:
  proxy-net:
    driver: bridge
```

### Step 3: Create `dynamic.yml`

Create a `dynamic.yml` file in the same directory to handle HTTP→HTTPS redirection and security headers:

```yaml
http:
  middlewares:
    redirect-to-https:
      redirectScheme:
        scheme: https
        permanent: true
    secure-headers:
      headers:
        sslRedirect: true
        forceSTSHeader: true
        stsIncludeSubdomains: true
        stsPreload: true
        stsSeconds: 31536000
        contentTypeNosniff: true
        browserXssFilter: true
        customFrameOptionsValue: SAMEORIGIN
        customResponseHeaders:
          X-Robots-Tag: "noindex, nofollow"  # Hides panel from search engines

  routers:
    http-catchall:
      rule: "HostRegexp(`{host:.+}`)"
      entryPoints:
        - "web"
      middlewares:
        - "redirect-to-https"
      service: "noop@internal"

tls:
  options:
    default:
      minVersion: VersionTLS12
      cipherSuites:
        - TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
        - TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
        - TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
        - TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
        - TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305
        - TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305
```

### Step 4: Apply Changes

1. Replace the placeholders in `docker-compose.yml`:
    - `YOUR_EMAIL@example.com` → your actual email
    - `your_cloudflare_api_token_here` → your Cloudflare API token
    - `panel.yourdomain.com` → your actual domain
2. In Cloudflare, ensure the DNS record for `panel.yourdomain.com` points to your server and is set to **DNS Only** (grey cloud, not proxied).
3. Start everything:

    ```bash
    docker compose up -d
    ```
