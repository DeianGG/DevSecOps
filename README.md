# DevSecOps WordPress Lab

Automated vulnerability discovery and remediation pipeline for a containerized WordPress deployment. Combines **Docker Compose**, **WPScan**, and **GitHub Actions** to demonstrate a shift-left security workflow: deploy → scan → analyze → remediate → rebuild → re-scan.

> **Course project** — DevSecOps module. This lab is intentionally insecure in its baseline form. Do not deploy the `vulnerable` image to the public internet.

---

## Results at a glance

| Metric | Baseline (`wordpress:5.7.0`) | Hardened (`deiangg/devsecops:hardened`) |
|---|---|---|
| WordPress core CVEs | **46** | **0** |
| Vulnerable plugins | 1 (Akismet) | 0 (plugin removed) |
| User enumeration (`?author=N`, REST) | `admin` leaked | Blocked (403) |
| XML-RPC endpoint | Enabled | Blocked (403) |
| `readme.html` / `license.txt` | Exposed | Blocked / removed |
| `Server` header | `Apache/2.4.38 (Debian)` | `Apache` |
| `X-Powered-By` | `PHP/7.4.16` | Removed |
| Security response headers | None | `X-Content-Type-Options`, `X-Frame-Options`, `Referrer-Policy`, `Permissions-Policy`, `X-XSS-Protection` |

Full scan reports are committed under [`scans/`](scans/).

---

## Repository layout

```
DevSecOps/
├── README.md
├── docker/
│   ├── docker-compose.yml              # vulnerable baseline (WordPress 5.7.0 + MariaDB)
│   ├── docker-compose.hardened.yml     # hardened deployment (uses the built image)
│   ├── Dockerfile                      # hardened image definition
│   ├── hardening/
│   │   ├── security.ini                # PHP hardening overrides
│   │   └── security.conf               # Apache hardening: headers, blocks, rewrites
│   └── .env.example                    # copy to .env and edit
├── scans/
│   ├── before.txt                      # WPScan report against vulnerable baseline
│   └── after.txt                       # WPScan report against hardened image
├── src/
│   └── notes.txt
└── .github/workflows/
    ├── scan.yml                        # CI: spin up WP and run WPScan
    └── build-and-push.yml              # CI: build hardened image, push to Docker Hub + GHCR
```

---

## Quick start

### Prerequisites

- Docker Desktop (Docker Engine 24+) with `docker compose` plugin
- A free WPScan API token (https://wpscan.com/register) for full CVE detail (25 req/day on the free tier)

### 1. Bring up the vulnerable baseline

```bash
cd docker
cp .env.example .env
docker compose up -d
```

WordPress is exposed on http://localhost:8090. Complete the installer manually, or POST to it:

```bash
curl -s -X POST "http://localhost:8090/wp-admin/install.php?step=2" \
  --data-urlencode "weblog_title=DevSecOps Lab" \
  --data-urlencode "user_name=admin" \
  --data-urlencode "admin_password=Insecure123!Lab" \
  --data-urlencode "admin_password2=Insecure123!Lab" \
  --data-urlencode "pw_weak=1" \
  --data-urlencode "admin_email=admin@devsecops.lab" \
  --data-urlencode "Submit=Install WordPress"
```

### 2. Scan it locally with WPScan

```bash
export WPSCAN_API_TOKEN=<your-token>

docker run --rm --platform linux/amd64 \
  -v "$(pwd)/scans:/scans" \
  wpscanteam/wpscan \
  --url http://host.docker.internal:8090 \
  --api-token "$WPSCAN_API_TOKEN" \
  --enumerate u,vp,vt,cb,dbe \
  --plugins-detection aggressive \
  --random-user-agent \
  --format cli-no-color \
  -o /scans/before.txt
```

Inspect `scans/before.txt` — expect **46 WordPress core vulnerabilities** plus user enumeration, XML-RPC, and version-disclosure findings.

### 3. Run the hardened image

```bash
docker compose -f docker-compose.hardened.yml up -d
# Exposed on http://localhost:8091
```

Re-run the WPScan command above against `http://host.docker.internal:8091`, outputting to `/scans/after.txt`. Compare:

```bash
diff <(grep '\[!\] Title:' scans/before.txt) <(grep '\[!\] Title:' scans/after.txt)
```

---

## Hardening summary

The hardened Dockerfile applies the following changes to `wordpress:6.8.3-php8.3-apache`:

| Area | Mitigation |
|---|---|
| **Patch level** | Bumped from WordPress 5.7.0 / PHP 7.4 → WordPress 6.8.3 / PHP 8.3 (closes all 46 core CVEs) |
| **Surface reduction** | Removed `readme.html`, `license.txt`, `hello.php`, Akismet plugin; deleted all default themes except `twentytwentyfour` |
| **Apache** | `ServerTokens Prod`, `ServerSignature Off`, `TraceEnable Off`; removed `Server` version + `X-Powered-By` |
| **HTTP security headers** | `X-Content-Type-Options`, `X-Frame-Options: SAMEORIGIN`, `Referrer-Policy`, `Permissions-Policy`, `X-XSS-Protection` |
| **PHP** | `expose_php=Off`, `allow_url_fopen=Off`, `allow_url_include=Off`, strict session cookies, dangerous functions disabled (`exec`, `shell_exec`, `system`, `proc_open`, `popen`) |
| **XML-RPC** | Blocked at Apache (`Require all denied`) — closes DoS and credential brute-force vectors |
| **wp-config.php** | Direct access blocked (defence in depth) |
| **Dotfiles** | `.git`, `.htaccess`, `.env` blocked from web access |
| **Author enumeration** | `?author=<id>` query rewritten to 403 |
| **Uploads dir** | PHP/PHAR execution disabled in `wp-content/uploads/` (web-shell mitigation) |
| **WordPress constants** | `DISALLOW_FILE_EDIT`, `DISALLOW_FILE_MODS`, `WP_DEBUG=false`, `WP_POST_REVISIONS=5` |
| **File permissions** | All WP files `0644`, directories `0755`, owned by `www-data` |
| **OS packages** | Removed `wget`, `nano`, `vim-tiny`, `less` |
| **Non-root container** | Apache rebound to unprivileged port 8080; container runs entirely as `www-data` (UID 33) |
| **Observability** | Healthcheck against `/wp-login.php` |
| **Provenance** | OCI image labels for repo source + license |

---

## CI / CD pipeline

Two GitHub Actions workflows live under `.github/workflows/`:

### `scan.yml` — WPScan scanner
- **Triggers:** push to `main`, pull requests, manual (`workflow_dispatch`) with `wp_image` + `scan_label` inputs
- **What it does:**
  1. Spins up MariaDB + the target WordPress image
  2. Waits for the installer, runs the auto-install POST
  3. Runs WPScan with the API token from `secrets.WPSCAN_API_TOKEN`
  4. Uploads the report as an Actions artifact and (on `main`) commits it to `scans/`

### `build-and-push.yml` — Build & publish hardened image
- **Triggers:** changes under `docker/Dockerfile` or `docker/hardening/**`, plus manual
- **What it does:**
  1. Builds the hardened image multi-arch (`linux/amd64`, `linux/arm64`)
  2. Pushes to both **Docker Hub** (`deiangg/devsecops`) and **GitHub Container Registry** (`ghcr.io/deiangg/devsecops`)
  3. Triggers `scan.yml` with `wp_image` pointing at the freshly pushed image, completing the **scan → fix → re-scan loop**

### Required GitHub secrets

| Secret | How to obtain |
|---|---|
| `WPSCAN_API_TOKEN` | https://wpscan.com/register → Profile → API Token |
| `DOCKERHUB_TOKEN` | https://hub.docker.com → Account Settings → Security → New Access Token (Read, Write, Delete) |

---

## Published images

| Registry | Reference |
|---|---|
| Docker Hub | `docker.io/deiangg/devsecops:hardened` |
| GitHub Container Registry | `ghcr.io/deiangg/devsecops:hardened` |

Pull and run:
```bash
docker run -d --name wp-hardened \
  -p 8091:8080 \
  -e WORDPRESS_DB_HOST=<host> \
  -e WORDPRESS_DB_USER=<user> \
  -e WORDPRESS_DB_PASSWORD=<pass> \
  -e WORDPRESS_DB_NAME=wordpress \
  deiangg/devsecops:hardened
```

---

## DevSecOps strategy (shift-left)

This pipeline embeds security checks **at commit time**, not after deployment:

1. **Scan-as-CI:** every push to `main` triggers a fresh deploy + scan in a clean ephemeral environment.
2. **Versioned evidence:** WPScan reports live in the repo (`scans/`), giving an auditable history of the security posture.
3. **Continuous re-scan loop:** building a new hardened image automatically dispatches a re-scan, so a regression in the Dockerfile cannot ship silently.
4. **Provenance & multi-registry:** images are signed with OCI metadata and mirrored to two registries to avoid single-vendor lock-in.
5. **Reproducible baseline:** pinning the vulnerable image (`wordpress:5.7.0-php7.4-apache`) ensures the "before" findings are deterministic, so the diff against "after" is meaningful.

---

## Known limitations

- **WPScan free tier** caps at 25 API requests/day — sufficient for a few CI runs but not heavy iteration.
- **Container still runs Apache as root** (it must, to bind port 80, then drops privileges to `www-data` per Apache's MPM model). A fully rootless image would require either `php-fpm + nginx` or port remapping; documented as a known trade-off.
- **External `wp-cron.php`** remains reachable (60% confidence finding). For production, set `DISABLE_WP_CRON=true` and run a real system cron — left enabled here to keep the lab one-command.
- The **MariaDB credentials** in `.env.example` are weak by design (lab values). Rotate before any real deployment.

---

## License

MIT
