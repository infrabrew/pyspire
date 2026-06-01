# Deploying PySpire with systemd (port 8081)

PySpire is a static site - `index.html` plus the `css/` and `js/` folders - so
"deployment" is just serving those files. Two backends are provided; pick one.

| Source unit | Backend | Best for |
|---|---|---|
| `pyspire-python.service` | `python3 -m http.server` | Internal / low-traffic. Zero dependencies. |
| `pyspire-caddy.service` + `Caddyfile` | Caddy | Public / higher-traffic. gzip/zstd, cache + security headers. |

Both serve on **port 8081** from **`/var/www/pyspire`**, and both install to the
same runtime name `pyspire.service`. Because they share that name and port, only
one can be active at a time - which is exactly what you want. To switch backends,
copy the other source file over `/etc/systemd/system/pyspire.service` and run
`sudo systemctl restart pyspire.service`.

## 1. Copy the site into place (same for both)

```sh
sudo mkdir -p /var/www/pyspire
sudo cp -r index.html css js /var/www/pyspire/
```

## 2a. Option A - Python (no dependencies)

```sh
sudo cp deploy/pyspire-python.service /etc/systemd/system/pyspire.service
sudo systemctl daemon-reload
sudo systemctl enable --now pyspire.service
```

## 2b. Option B - Caddy (compression + headers)

```sh
sudo apt install caddy          # or: dnf install caddy / pacman -S caddy
sudo mkdir -p /etc/pyspire
sudo cp deploy/Caddyfile /etc/pyspire/Caddyfile
sudo cp deploy/pyspire-caddy.service /etc/systemd/system/pyspire.service
sudo systemctl daemon-reload
sudo systemctl enable --now pyspire.service
```

## 3. Verify (either option)

```sh
systemctl status pyspire.service
curl -I http://localhost:8081/        # expect 200 OK
journalctl -u pyspire.service -f      # live log
```

With Caddy you can also confirm compression is active:

```sh
curl -sI -H 'Accept-Encoding: gzip' http://localhost:8081/css/styles.css \
  | grep -i content-encoding          # expect: content-encoding: gzip
```

## Updating the site later

```sh
sudo cp -r index.html css js /var/www/pyspire/
sudo systemctl restart pyspire.service
```

## How the two compare

The difference is entirely in the response, not the content. Measured locally,
Python's http.server returns:

```
HTTP/1.0 200 OK
Server: SimpleHTTP/0.6 Python/3.x
Content-type: text/css
Content-Length: 101768
```

No `Content-Encoding` (no gzip - the ~100KB CSS ships uncompressed), no
`Cache-Control` (re-validated every visit), and a version banner in `Server:`.
Caddy fixes all three: compresses text assets (~100KB CSS becomes ~25KB), sets
`Cache-Control` on assets (1-day `max-age`; see the Caddyfile note on why not
`immutable`) while keeping HTML fresh, strips the server banner, and adds
`X-Content-Type-Options` / `X-Frame-Options` / `Referrer-Policy` /
`Permissions-Policy`.

## Notes & caveats

- **python3 path** - `pyspire-python.service` uses `/usr/bin/python3`. Confirm
  with `command -v python3` and edit `ExecStart` if yours differs.
- **caddy user** - `pyspire-caddy.service` runs as the `caddy` user/group, which
  the distro caddy package creates. If you installed Caddy manually, create that
  user or change `User=`/`Group=` to one that exists.
- **Reverse proxy** - for either backend behind nginx/another proxy, change the
  bind address to `127.0.0.1` (Python: `--bind 127.0.0.1`; Caddy: `127.0.0.1:8081`)
  so only the proxy talks to the app.
- **Security** - both units are sandboxed (`ProtectSystem=strict`, read-only site
  directory, restricted syscalls/namespaces). The site's Content-Security-Policy
  ships in the `<meta>` tag inside `index.html`, so it applies under either server.
- **Firewall** - if the box runs a firewall, open the port:
  `sudo ufw allow 8081/tcp` (or the firewalld equivalent).

## Raspberry Pi 4 + Cloudflare Tunnel (Caddy backend)

Recommended for exposing PySpire publicly from a Pi without opening any ports.

```
Browser --HTTPS--> Cloudflare edge --tunnel--> cloudflared (Pi) --HTTP--> 127.0.0.1:8081 (Caddy) --> files
```

Cloudflare's edge terminates TLS, serves Brotli, and caches at the CDN. The Pi
serves plain HTTP on loopback only. **No inbound ports** on the Pi or router --
cloudflared dials OUT to Cloudflare.

### What changes vs. a directly-exposed Caddy

- **Caddy binds `127.0.0.1:8081`, not `:8081`** (already set in `Caddyfile`). The
  tunnel is the only client; nothing else can reach the origin.
- **Keep Caddy on plain HTTP.** Do not give Caddy a domain for auto-HTTPS -- TLS
  is Cloudflare's job. Two TLS layers would conflict.
- **Compression is largely redundant** -- Cloudflare serves Brotli at the edge;
  the cloudflared<->Caddy hop is loopback. Caddy's real remaining value here is
  the **Cache-Control headers** (they drive Cloudflare's edge caching) plus the
  security headers and robust multi-threaded serving.

### Install

1. Deploy the site + Caddy as in **Option B** above (Caddy already binds
   localhost via the provided `Caddyfile`).
2. Install cloudflared (ARM64 build for Pi OS 64-bit) per Cloudflare's current
   docs: <https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/>
3. Create the tunnel and map a public hostname to it, then either:
   - **Dashboard (token) tunnel:** run
     `sudo cloudflared service install <TOKEN>` -- ingress is managed in the
     Cloudflare dashboard (point it at `http://127.0.0.1:8081`). Simplest.
   - **Locally-managed tunnel:** copy `deploy/cloudflared-config.yml` to
     `/etc/cloudflared/config.yml`, fill in your tunnel ID + credentials path and
     hostname, then `sudo cloudflared service install`.
4. `sudo systemctl enable --now cloudflared` (and `pyspire.service` for Caddy).

### Verify

```sh
curl -I http://127.0.0.1:8081/                 # on the Pi: origin is up (200)
curl -I https://pyspire.example.com/           # anywhere: tunnel works (200)
systemctl status cloudflared pyspire.service
```

### Deploy footgun: stale cache

Asset filenames are stable, so after a deploy both browsers and Cloudflare may
serve the old `app.js`/`styles.css`. The `Caddyfile` uses a 1-day `max-age` (no
`immutable`) to bound this. For instant propagation, **purge the Cloudflare cache
on deploy** (dashboard or API), or switch to content-hashed filenames and raise
the cache lifetime.
