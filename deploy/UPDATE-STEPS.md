# Updating the PySpire site (minified deploy to 192.168.0.2)

The live site at **192.168.0.2** (nginx, port 80, web root `/var/www/html`)
serves the **minified** build from `dist/`. Your readable source stays in the
repo root (`index.html`, `css/styles.css`, `js/app.js`) — that's what you edit.

Whenever you change a source file, follow these steps to push the update.

---

## Step 1 — Edit the readable source

Edit the originals in the repo root, NOT the files in `dist/`:

- `index.html`
- `css/styles.css`
- `js/app.js`

(`dist/` is overwritten by the build, so any manual edit there is lost.)

## Step 2 — Build (minify)

From the repo root:

```sh
cd /Volumes/share/pyspire
./build.sh
```

This regenerates `dist/index.html`, `dist/css/styles.css`, `dist/js/app.js`
(minified) and prints the size savings. Requires `node`/`npx` (already installed).

## Step 3 — Copy the built files to the server

```sh
scp dist/index.html dist/css/styles.css dist/js/app.js coreai@192.168.0.2:/tmp/
```

## Step 4 — Back up the current site, then install the new files

One SSH command does backup + install + on-disk verify:

```sh
ssh coreai@192.168.0.2 '
set -e
WEBROOT=/var/www/html
TS=$(date +%Y%m%d-%H%M%S)
sudo mkdir -p /var/backups/pyspire
sudo tar -czf /var/backups/pyspire/html-$TS.tar.gz -C "$WEBROOT" index.html css js
echo "backup: /var/backups/pyspire/html-$TS.tar.gz"
sudo install -d -m 755 "$WEBROOT/css" "$WEBROOT/js"
sudo install -m 644 /tmp/index.html "$WEBROOT/index.html"
sudo install -m 644 /tmp/styles.css "$WEBROOT/css/styles.css"
sudo install -m 644 /tmp/app.js     "$WEBROOT/js/app.js"
rm -f /tmp/index.html /tmp/styles.css /tmp/app.js
echo "== on-disk sha256 =="
sudo sha256sum "$WEBROOT/index.html" "$WEBROOT/css/styles.css" "$WEBROOT/js/app.js"
'
```

## Step 5 — Verify what the server actually serves

Compare served files to your local `dist/` (a cache-busting query is added so
you never compare against a stale browser/proxy copy):

```sh
cd /Volumes/share/pyspire
for f in index.html css/styles.css js/app.js; do
  R=$(curl -s "http://192.168.0.2/$f?v=$(date +%s)$RANDOM" | shasum -a 256 | cut -d' ' -f1)
  L=$(shasum -a 256 "dist/$f" | cut -d' ' -f1)
  [ "$R" = "$L" ] && echo "$f OK" || echo "$f MISMATCH"
done
```

All three should print `OK`. Then hard-refresh your browser (Cmd+Shift+R) to see
the change.

---

## Rollback (if an update breaks the site)

Backups are timestamped tarballs in `/var/backups/pyspire/` on the box.

```sh
ssh coreai@192.168.0.2 '
ls -t /var/backups/pyspire/                       # newest first
LATEST=$(ls -t /var/backups/pyspire/html-*.tar.gz | head -1)
echo "restoring $LATEST"
sudo tar -xzf "$LATEST" -C /var/www/html
'
```

## If the site does not respond at all

nginx may be stopped. Check and start it:

```sh
ssh coreai@192.168.0.2 'systemctl is-active nginx || sudo systemctl start nginx'
```

---

## Notes

- **Minification makes code unreadable, not secret, and does NOT hide your IP.**
  The browser still connects to 192.168.0.2 (visible in DevTools > Network,
  `ping`, etc.). To hide the origin IP you need a proxy/tunnel — see the
  Cloudflare Tunnel notes in `deploy/README.md`.
- **Debugging is harder against minified code.** To temporarily serve the
  readable version, copy the repo-root files instead of the `dist/` ones in
  Steps 3-4 (`scp index.html css/styles.css js/app.js ...`).
- **One-time SSH note:** rapid repeated SSH connects can trip fail2ban and
  briefly block your IP. If SSH/HTTP suddenly refuses, wait ~10 min and retry.
