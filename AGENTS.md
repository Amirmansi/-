# erpnext_com

`erpnext_com` is the **ERPNext.com marketing website**, packaged as a [Frappe Framework](https://frappeframework.com) app (it is not a standalone server). It is installed into a Frappe **bench** + site, which provides the web server, MariaDB, Redis, the realtime (socketio) service and background workers.

## Cursor Cloud specific instructions

The environment is already provisioned in the VM snapshot. The heavy, one-time setup (system packages, the Frappe bench, the site, and the installed apps) is NOT repeated by the update script, so the notes below are what you need to actually run and test things.

### Layout
- Bench lives at `~/frappe-bench` (outside the repo). `bench` is installed in `~/.local/bin` (already on `PATH` via `~/.bashrc`).
- This repo is symlinked into the bench as the app: `~/frappe-bench/apps/erpnext_com -> /workspace`. Editing files in `/workspace` directly affects the running app.
- Site name: `erpnext.localhost` (default site). Desk login: `Administrator` / `admin`. MariaDB `root` password: `frappe`.
- `frappe_theme` (public app `frappe/frappe_theme`, `master` branch) is installed alongside `erpnext_com`. Several marketing pages (`/healthcare`, `/manufacturing`, `/retail`, `/education`, `/agriculture`, `/services`, `/contact`, `/distribution`, `/non-profit`, `/india-gst`) `{% extends "frappe_theme/templates/base.html" %}` and return HTTP 500/417 without it. `/about` uses the core base template and works without it.

### Frappe version note
The app is from the Frappe v12/v13 era but runs on **Frappe `version-15`** (the only line compatible with the VM's Python 3.12). Deprecated `hooks.py` keys (`app_color`, `app_icon`) are ignored by v15. Treat the repo code as correct/working — do not "modernize" it.

### Starting services (must be done each fresh VM; no systemd here, PID 1 is `tini`)
Run these before testing — they do NOT auto-start on snapshot restore:
```bash
sudo service mariadb start
sudo service redis-server start   # optional; bench starts its own redis on :13000/:11000
cd ~/frappe-bench && bench start   # web :8000, socketio :9000, workers, watcher, redis
```
`bench start` runs the whole dev stack via honcho (Procfile). Prefer running it in a tmux session so it survives. The site is served at `http://erpnext.localhost:8000` (`erpnext.localhost` is mapped to `127.0.0.1` in `/etc/hosts`). The desk is at `/app`.

### Gotchas
- After adding/installing a NEW app or changing `sites/apps.txt`, the running `bench start` web process holds a stale module map and every page 500s with `'NoneType' object has no attribute 'get'`. Fully restart `bench start` (a hot reload is not enough). Editing existing Python/JS/templates is picked up by the watcher/reloader fine.
- Rebuild assets with `bench build` (or `bench build --app <app>`); the bench was initialized with `--skip-assets`, so assets are built post-init.
- `erpnext_com/api.py` imports the proprietary `central` app and is only used by the conference payment flow; it is imported lazily, so app install and normal page serving work without `central`. Payment endpoints (`make_payment`) will fail without `central` + Razorpay/PayPal config.
- `erpnext_com/utils.get_country` calls `pro.ip-api.com` and needs `ip-api-key` in site config; it only matters for geo-based pricing.
- A brand-new Frappe site sends the first desk login through the Setup Wizard. This site is already marked complete (persisted in the snapshot DB), so `Administrator`/`admin` lands directly on the desk. In Frappe v15 the relevant flag is `Installed Application.is_setup_complete` for the `frappe` app (NOT `System Settings.setup_complete`). If the wizard ever reappears, in `bench --site erpnext.localhost console` run: `frappe.db.set_value("Installed Application", n, "is_setup_complete", 1)` for the `frappe` row, then `frappe.db.commit(); frappe.clear_cache()`. Also clear the browser's cookies/session, since a stale logged-in session keeps the cached boot.

### Lint / test / build / run quick reference (from `~/frappe-bench`)
- Run: `bench start`
- Build assets: `bench build`
- App tests (Frappe test runner): `bench --site erpnext.localhost run-tests --app erpnext_com` (the app ships no real unit tests, so this is a near no-op).
- Spell check (the repo's only CI, see `.github/workflows/nodejs.yml`): `npx cspell --config ./spell_checker/cSpell.json <files>`.
- Create/inspect records: `bench --site erpnext.localhost console` or `bench --site erpnext.localhost execute <dotted.path>`.
