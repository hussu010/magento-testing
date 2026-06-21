# magento-testing

A local Docker environment for testing the **Stripe Payments** extension against
Magento. Built on top of [markshust/docker-magento](https://github.com/markshust/docker-magento).

This repo contains **only the harness** (Docker config + `bin/` helper scripts).
The Magento source code lives in `src/` and is **not committed** (see
[`.gitignore`](.gitignore)) — you rebuild it locally with the steps below.

---

## What's in this repo

| Path | Purpose |
|------|---------|
| `compose*.yaml` | Docker Compose service definitions |
| `bin/` | Helper scripts (`bin/setup`, `bin/download`, `bin/magento`, …) |
| `bin/upgrade-stripe` | **Custom** — install/upgrade the Stripe module from a `.tgz` ([guide](docs/UPGRADING_STRIPE.md)) |
| `env/` | Environment variables (DB creds, admin user, etc.) |
| `docs/` | Project documentation |
| `src/` | Magento install — **git-ignored**, generated locally |

> `compose.dev-ssh.yaml` and `compose.dev-linux.yaml` are optional overrides
> (SSH-agent forwarding / Linux). They are **not** auto-loaded on macOS and can
> be ignored unless you need them.

---

## Prerequisites

- **Docker Desktop** with **≥ 6 GB RAM** allocated (the setup aborts otherwise)
- macOS or Linux
- `~5 GB` free disk

---

## Quick start (from a clean clone)

```bash
# 1. Download the Magento source.
#    We pin Mage-OS 1.0.6 (= Magento 2.4.7-p4) on purpose — see "Why Mage-OS 1.0.6" below.
bin/download mageos 1.0.6

# 2. Install & configure Magento at https://magento.test/
#    Run this in a real terminal — it will prompt for your sudo password
#    to add `magento.test` to /etc/hosts and to trust the local SSL CA.
bin/setup magento.test
```

When `bin/setup` finishes, the store is live at **https://magento.test/**.

> **If you ran `bin/setup` over a non-interactive shell** (no TTY), it aborts at
> the `/etc/hosts` step. Finish manually:
> ```bash
> echo "127.0.0.1 ::1 magento.test" | sudo tee -a /etc/hosts   # required
> bin/setup-ssl-ca                                             # optional: trust the SSL cert
> ```

### 3. Install the Stripe extension

Download the module `.tgz` (Adobe Marketplace, or the public
[stripe/stripe-magento2-releases](https://github.com/stripe/stripe-magento2-releases)
GitHub repo), then:

```bash
bin/upgrade-stripe ~/Downloads/stripe-magento2-X.Y.Z.tgz
```

This extracts the module into `src/`, requires the matching `stripe/stripe-php`
SDK, enables the modules, and runs `setup:upgrade` + `di:compile`. See
[docs/UPGRADING_STRIPE.md](docs/UPGRADING_STRIPE.md) for details and the manual steps.

### 4. (local dev) Disable 2FA

Magento forces Two-Factor Auth on the admin. For local testing, disable it:

```bash
bin/clinotty bin/magento module:disable Magento_TwoFactorAuth
bin/clinotty bin/magento setup:upgrade
bin/clinotty bin/magento setup:di:compile
bin/clinotty bin/magento cache:flush
```

### 5. (optional) Load Luma sample data

Gives you ~2,000 products to test checkout against:

```bash
bin/clinotty bin/magento sampledata:deploy
bin/clinotty bin/magento setup:upgrade
bin/clinotty bin/magento setup:di:compile
bin/clinotty bin/magento indexer:reindex
bin/clinotty bin/magento cache:flush
```

---

## Access

| | |
|---|---|
| **Storefront** | https://magento.test/ |
| **Admin** | https://magento.test/admin/ |
| **Admin user** | `john.smith` |
| **Admin password** | `password123` |

Credentials are defined in [`env/magento.env`](env/magento.env).

---

## Testing Stripe

1. Admin → **Stores → Configuration → Sales → Payment Methods → Stripe**.
2. Enter your **test-mode** keys (`pk_test_…` / `sk_test_…`) from the
   [Stripe Dashboard](https://dashboard.stripe.com) → Developers → API keys.
   Set **Enabled = Yes** and save. An invalid key errors on save — the first
   sign the module is reaching Stripe's API.
3. Add a product → checkout → choose Stripe → use test card
   `4242 4242 4242 4242`, any future expiry, any CVC/ZIP.
4. Confirm the order is paid in **Sales → Orders** *and* the charge appears in
   the Stripe Dashboard → **Payments** (test mode).

---

## Why Mage-OS 1.0.6 (not the default 3.0.0)?

`bin/download mageos` defaults to **3.0.0**, but its `magento2-base` dist zip on
`repo.mage-os.org` has a **corrupt checksum** — every `composer install` fails
with *"The checksum verification of the file failed"*. Versions **1.0.6** and
**2.3.0** are intact. We use **1.0.6** because it maps to **Magento 2.4.7-p4**,
the platform the older Stripe `stripe-payments` 4.6.x module targets.

To verify a version before committing to it:

```bash
curl -s https://repo.mage-os.org/p2/mage-os/magento2-base.json \
  | python3 -c "import sys,json;d=json.load(sys.stdin);[print(p['version'],p['dist']['shasum']) for p in d['packages']['mage-os/magento2-base']]"
# then compare the shasum against `shasum` of the downloaded zip
```

---

## Common commands

```bash
bin/start            # start containers
bin/stop             # stop containers
bin/restart          # restart containers
bin/clinotty bin/magento <cmd>   # run a Magento CLI command
bin/magento <cmd>    # shortcut for the above (TTY)
bin/cache-clean      # clear caches
bin/bash             # shell into the phpfpm container
bin/mysql            # open a MySQL shell
bin/removeall        # tear everything down (containers + volumes)
```

---

## Troubleshooting

- **`chmod: cannot access 'bin/magento'`** during `bin/setup` — you skipped
  `bin/download`. Run `bin/download mageos 1.0.6` first.
- **Checksum verification failed** during download — you're on a bad Mage-OS
  version (e.g. 3.0.0). Use 1.0.6 (see above).
- **Browser "Your connection is not private"** — the local SSL cert isn't
  trusted yet. Run `bin/setup-ssl-ca`, then **fully quit and reopen** the
  browser (Cmd+Q). If the served cert and the trusted CA drift apart (after
  container recreation), re-issue the cert with `bin/setup-ssl magento.test`.
- **Admin keeps asking for 2FA** — disable it for local dev (step 4 above).
