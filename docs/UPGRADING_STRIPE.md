# Installing / Upgrading the Stripe extension

This environment installs the Stripe Magento module from a release **`.tgz`**
(an `app/code`-style package), **not** via `composer require stripe/stripe-payments`.

> **Why the `.tgz` method?** The composer package `stripe/stripe-payments` is
> served from `repo.magento.com`, which needs paid Adobe Marketplace auth keys.
> This repo intentionally uses Mage-OS (no keys), so we install the module's
> source directly. The `stripe/stripe-php` SDK it depends on is on public
> Packagist, so that part still uses Composer.

## Where to get the module

- **Adobe Commerce Marketplace** — download the extension package, or
- **Public GitHub releases**:
  [stripe/stripe-magento2-releases](https://github.com/stripe/stripe-magento2-releases)
  — pick a tagged release (e.g. `stripe-magento2-X.Y.Z.tgz`) or
  `stripe-magento2-latest.tgz`.

---

## The easy way: `bin/upgrade-stripe`

```bash
bin/upgrade-stripe ~/Downloads/stripe-magento2-X.Y.Z.tgz
```

The script:
1. Removes the existing `src/app/code/StripeIntegration` (clean upgrade).
2. Extracts the new `.tgz` into `src/`.
3. Auto-detects the required `stripe/stripe-php` version from the module docs
   and runs `composer require` for it.
4. Syncs to the container, enables the modules, and runs
   `setup:upgrade` → `setup:di:compile` → `cache:flush` → `fixowns`/`fixperms`.
5. Prints the new module version.

This works for both a **first install** and an **upgrade**.

---

## The manual way

Use this if you want to understand each step or the script fails.

### 1. Check the required SDK version

The module pins a `stripe/stripe-php` range. Find it in the package docs:

```bash
# after extracting, or from inside the tgz:
grep -rEo "composer require stripe/stripe-php:[^ ]+" \
  src/app/code/StripeIntegration/Payments/resources/docs/payments/install.md
# e.g. -> composer require stripe/stripe-php:~20.1
```

### 2. Replace the module source

```bash
rm -rf src/app/code/StripeIntegration          # remove the old version
tar -xzf ~/Downloads/stripe-magento2-X.Y.Z.tgz -C src/
bin/copytocontainer --all                       # sync into the container
```

### 3. Require the SDK (in the container)

```bash
bin/clinotty composer require stripe/stripe-php:~20.1   # use the constraint from step 1
bin/copyfromcontainer composer.json                     # sync lock changes back to host
bin/copyfromcontainer composer.lock
```

### 4. Enable + apply

```bash
bin/clinotty bin/magento module:enable StripeIntegration_Tax StripeIntegration_Payments
bin/clinotty bin/magento setup:upgrade
bin/clinotty bin/magento setup:di:compile
bin/clinotty bin/magento cache:flush
bin/fixowns
bin/fixperms
```

### 5. Verify

```bash
bin/clinotty bin/magento module:status StripeIntegration_Payments   # -> enabled
cat src/app/code/StripeIntegration/Payments/VERSION.txt             # new version
bin/clinotty cat vendor/stripe/stripe-php/VERSION                   # SDK version
```

Then re-enter test API keys in the admin if needed and run a test checkout
(see the [README](../README.md#testing-stripe)).

---

## Removing the module

```bash
bin/clinotty bin/magento module:disable StripeIntegration_Payments StripeIntegration_Tax
rm -rf src/app/code/StripeIntegration
bin/clinotty composer remove stripe/stripe-php
bin/clinotty bin/magento setup:upgrade
bin/clinotty bin/magento setup:di:compile
bin/clinotty bin/magento cache:flush
```

---

## Notes

- After an upgrade, in **production mode** also run
  `bin/clinotty bin/magento setup:static-content:deploy -f`. In developer mode
  (this env's default) it's not required.
- Check the module's `CHANGELOG.md` for breaking changes and any new
  `stripe/stripe-php` requirement before upgrading.
- If a new module version needs a newer SDK, step 3 (`composer require
  stripe/stripe-php:<new constraint>`) is the part that changes.
