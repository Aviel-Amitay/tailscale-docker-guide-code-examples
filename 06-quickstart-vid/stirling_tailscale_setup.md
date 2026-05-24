# Stirling PDF with Tailscale Docker Setup

This guide explains how to run Stirling PDF behind Tailscale using Docker Compose.

## Table tree
```
├── 06-quickstart-vid
│   ├── 01-nginx-basic
│   │   └── docker-compose.yaml
│   ├── 02-stirlingpdf
│   │   ├── config
│   │   │   └── stirling.json
│   │   └── docker-compose.yaml
│   ├── stirling_tailscale_setup.md
```

> Important: DON'T pasted a real Tailscale auth key into ChatGPT, GitHub, logs, or screenshots.  
In case you did, revoke it from the Tailscale admin console and generate a new one.

## 1. Generate a Tailscale Auth Key

Go to admin console https://login.tailscale.com/admin/settings/general

```text
Tailscale Admin Console -> Settings -> Keys -> Generate auth key
```

Recommended options:

```text
Reusable: enabled
Ephemeral: disabled
Pre-approved: enabled
Expiration: short, for example 1 to 7 days
```

You do not need an OAuth key for this setup. Use an Auth key.

## 2. Use a .env File

Create a `.env` file in the same directory as your `docker-compose.yaml`, to store your key:

```bash
TS_AUTHKEY=<tskey-auth-NEW_KEY_HERE>
```

Do not put the real key directly inside `docker-compose.yaml`.

## 3. Docker Compose Example

- Example of yaml file [docker-compose.yaml](./02-stirlingpdf/docker-compose.yaml):

```yamlservices:
  tailscale-authkey1:
    image: tailscale/tailscale:latest
    container_name: ts-authkey-test
    hostname: banana
    environment:
      - TS_AUTHKEY=${TS_AUTHKEY}
      - TS_STATE_DIR=/var/lib/tailscale
      - TS_USERSPACE=false
    volumes:
      - ts-authkey-test:/var/lib/tailscale
    devices:
      - /dev/net/tun:/dev/net/tun
    cap_add:
      - net_admin
    restart: unless-stopped
  nginx-authkey-test:
    image: nginx
    network_mode: service:tailscale-authkey1

# Stirling PDF services
  stirling-ts:
    image: tailscale/tailscale:latest
    container_name: stirling-ts
    hostname: pdf
    environment:
      - TS_AUTHKEY=${TS_AUTHKEY}
      - "TS_EXTRA_ARGS=--advertise-tags=tag:container --reset"
      - TS_SERVE_CONFIG=/config/stirling.json
      - TS_STATE_DIR=/var/lib/tailscale
      - TS_USERSPACE=false
    volumes:
      - ${PWD}/config:/config
      - stirling-ts:/var/lib/tailscale
    devices:
      - /dev/net/tun:/dev/net/tun
    cap_add:
      - net_admin
    restart: unless-stopped
  stirlingpdf:
    image: frooodle/s-pdf:latest
    container_name: stirlingpdf
    network_mode: service:stirling-ts
    depends_on:
      - stirling-ts
    volumes:
      - ./workspace:/workspace
      - stirling-config:/configs
      - stirling-storage:/usr/share/tesseract-ocr/5/tessdata
    environment:
      - DOCKER_ENABLE_SECURITY=false
    restart: unless-stopped

volumes: 
# Tailscale volumes
  ts-authkey-test:
    driver: local
# Stirling PDF volumes
  stirling-ts:
    driver: local
  stirling-config:
    driver: local
  stirling-storage:
    driver: local
```

Do not expose your Oauth key inside of the yaml file:

```yaml
environment:
  - TS_AUTHKEY=${TS_AUTHKEY}
```

## 4. Stirling Serve Config

Edit this file [stirling.json](./02-stirlingpdf/config/stirling.json):

```bash
nano config/stirling.json
```

- Replace the line `"${TS_CERT_DOMAIN}:443"` with your with your real Tailscale MagicDNS name.
 

```bash
sed -i '' 's/${TS_CERT_DOMAIN}/<hostname>.<YOUR-TAILSCAL-DNS>.ts.net/g' stirling.json
```
Example:

```json
{
  "TCP": {
    "443": {
      "HTTPS": true
    }
  },
  "Web": {
    "pdf.YOUR-TAILNET.ts.net:443": {
      "Handlers": {
        "/": {
          "Proxy": "http://127.0.0.1:8080"
        }
      }
    }
  },
  "AllowFunnel": {
    "pdf.YOUR-TAILNET.ts.net:443": false
  }
}
```



Do not leave this in the JSON file:

```json
"${TS_CERT_DOMAIN}:443"
```

A normal JSON file does not automatically replace `${TS_CERT_DOMAIN}` unless you have a script that generates the file from a template.

## 5. Verify Tailscale Connection

Check the Tailscale container logs:

```bash
docker logs stirling-ts
```

Check Tailscale VPN device status:

```bash
docker exec -it stirling-ts tailscale status
```

You should see the machine connected as:

```text
aviela@MacBook-M1 config % docker exec -it stirling-ts tailscale status

192.168.125.15  pdf                 pdf.magicDNS.ts.net  linux  -                          
192.168.132.88  banana              tailscale@           linux  -                          
10.16.86.61     iphone-15           tailscale@           iOS    offline, last seen 6d ago  
10.90.113.31     macbook-m1          tailscale@           macOS  idle, tx 340 rx 268   
```

## 6. Open Stirling PDF

Open this URL from your browser:

```text
https://pdf.YOUR-TAILNET.ts.net
```

Use the full Tailscale MagicDNS name, not `localhost`.

## Troubleshooting

### Restart the Containers

Run:

```bash
docker compose down
docker compose up -d
```

### Check if Stirling is running

```bash
docker logs stirlingpdf
```

### Check if Tailscale Serve loaded the config

```bash
docker logs stirling-ts
```

### Confirm containers are running

```bash
docker ps
```

### Common issue

If the URL does not open, confirm that `stirling.json` contains your real domain and not:

```text
${TS_CERT_DOMAIN}
```

## Notes About `?ephemeral=false`

You usually do not need to add this manually if you generated the key with ephemeral disabled.

Recommended `.env`:

```bash
TS_AUTHKEY=tskey-auth-NEW_KEY_HERE
```

@Write by https://github.com/Aviel-Amitay