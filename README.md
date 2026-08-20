# cdvrblack — dark mode for the Channels DVR admin UI

nginx runs as a second front door that proxies to CDVR and injects a dark
stylesheet into the HTML response.

- `http://<host>:8089` — unchanged, normal light-mode CDVR
- `http://<host>:8088` — same UI, dark mode, via nginx

No CDVR config changes. No firewall changes. Nothing rebinds.

## How it works

`nginx/channels-dvr-dark.conf` proxies everything on the dark-mode port
(above) to CDVR's own port, and uses nginx's `sub_filter` to insert
`<link rel="stylesheet" href="/__cdvr-dark-mode__.css">` right before
`</head>` in every HTML page. That CSS file
(`nginx/dark-mode.css`) is served directly by nginx from a fixed path.

The theme targets the app's actual Bootstrap 4 classes (`.card`,
`.form-control`, `.navbar`, `.dropdown-item`, `.btn-*`, `.table`,
`.modal-*`, etc.) rather than a blanket color-invert filter, so channel
logos, posters and thumbnails are left alone — only chrome/UI gets themed.

## Install (on the Channels DVR Linux host)

```bash
sudo cp nginx/channels-dvr-dark.conf /etc/nginx/conf.d/channels-dvr-dark.conf
sudo mkdir -p /etc/nginx/channels-dark
sudo cp nginx/dark-mode.css /etc/nginx/channels-dark/dark-mode.css
sudo nginx -t
sudo systemctl reload nginx
```

If nginx isn't installed yet:

```bash
sudo apt-get install nginx   # Debian/Ubuntu
# or
sudo dnf install nginx       # Fedora/RHEL
sudo systemctl enable --now nginx
```

Then browse to the dark-mode URL shown at the top of this file.

## Notes / things to check after installing

- **The dark-mode port must be free.** Change the `listen` line in the
  conf if something else already uses that port.
- **CDVR must already be listening on the same host** (its default port,
  out of the box). If your CDVR is on a different port (or scheme),
  update the `set $cdvr_address ...;` line — it feeds both `proxy_pass`
  and the HLS URL rewrite (see below), so it only needs changing in one
  place.
- Live status / recording-progress updates use WebSockets — the config
  forwards `Upgrade`/`Connection` headers for that, so those should keep
  working through the dark-mode port too.
- The theme was built from inspecting the live admin UI's real CSS classes
  (Bootstrap 4), not from an official CDVR dark theme — none exists upstream
  as of this writing. If a CDVR update changes its markup/classes, some
  elements may fall back to unstyled light colors until `dark-mode.css` is
  touched up — nothing will break functionally, since the plain light-mode
  URL keeps working regardless.
- Tabs/sections not visited during this build (e.g. `Live TV & DVR`,
  `Clients`, `Advanced`, `About`, and various modals) weren't individually
  checked — they use the same Bootstrap components so they should inherit
  the theme, but worth a quick look after installing.
- **Live TV/recording playback**: CDVR bakes its own real address into the
  HLS manifest it returns, which breaks playback through the dark-mode
  port. The config strips that down to a relative path via `sub_filter`,
  so the player's follow-up requests naturally come back through this
  proxy instead.
  This same variable is also what `proxy_pass` uses to actually reach
  CDVR. It's hardcoded to loopback for simplicity/reliability of that
  same-box hop — if you've configured CDVR (via its own `-host` flag) to
  not listen on loopback, or if you ever split nginx and CDVR onto
  different boxes/containers, change that line to CDVR's real reachable
  address instead. Scheme is included too, so a future CDVR change to
  HTTPS on its own port would only need changing here as well.
