# cdvrblack — dark mode for the Channels DVR admin UI

The Channels DVR binary actually does support binding to a specific address —
`channels-dvr -h` shows a `-host string` flag ("bind address for local web
interface") — despite the developers stating on the
[community forum](https://community.getchannels.com/t/dvr-network-interface-binding/33516)
that it always binds `0.0.0.0`. That forum thread is either outdated or wrong;
trust `-h` over it.

This project doesn't use `-host` though: rebinding CDVR to loopback and
making nginx the sole external door was considered and deliberately not
done (see git history / project discussion) — the simpler two-port setup
below was chosen instead. Revisit `-host 127.0.0.1` if you want CDVR
unreachable except through nginx.

- `http://<host>:8089` — unchanged, normal light-mode CDVR
- `http://<host>:8088` — same UI, dark mode, via nginx

No CDVR config changes. No firewall changes. Nothing rebinds.

## How it works

`nginx/channels-dvr-dark.conf` proxies everything on `:8088` to
`127.0.0.1:8089`, and uses nginx's `sub_filter` to insert
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

Then browse to `http://<host-ip>:8088`.

## Notes / things to check after installing

- **Port 8088 must be free.** Change the `listen 8088;` line in the conf if
  something else already uses it.
- **CDVR must already be listening on 8089 on the same host** (default). If
  your CDVR is on a different port, update `proxy_pass http://127.0.0.1:8089;`
  accordingly.
- Live status / recording-progress updates use WebSockets — the config
  forwards `Upgrade`/`Connection` headers for that, so those should keep
  working through the dark-mode port too.
- The theme was built from inspecting the live admin UI's real CSS classes
  (Bootstrap 4), not from an official CDVR dark theme — none exists upstream
  as of this writing. If a CDVR update changes its markup/classes, some
  elements may fall back to unstyled light colors until `dark-mode.css` is
  touched up — nothing will break functionally, since :8089 keeps working
  regardless.
- Tabs/sections not visited during this build (e.g. `Live TV & DVR`,
  `Clients`, `Advanced`, `About`, and various modals) weren't individually
  checked — they use the same Bootstrap components so they should inherit
  the theme, but worth a quick look after installing.
