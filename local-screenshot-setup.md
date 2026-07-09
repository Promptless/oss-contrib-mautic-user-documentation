# Running Mautic locally for documentation screenshots

Purpose: stand up the Mautic product on `localhost` at a **specific version**, log in,
and capture screenshots for these docs. Screenshots must match the version the docs
describe — Mautic's UI differs between minor releases (e.g. 7.0 vs 7.2), so always pin
the version before capturing.

Two setup paths are documented and both are verified:

- **Path A — Native, no Docker** (below). Runs PHP + MariaDB + the app as plain host
  processes. No Docker daemon, no root/sudo. This is the path an agent sandbox should
  use (bake the toolchain into the job image; do **not** attempt docker-in-docker).
- **Path B — DDEV** (near the bottom). One command, but requires Docker. Convenient for
  a human on a laptop; not suitable for a sandbox without a Docker daemon.

The product source lives in a **separate** repo (`github.com/mautic/mautic`), not in
this docs repo. Clone it somewhere outside this repo.

---

## 1. Pick the version (do this first)

```bash
git clone https://github.com/mautic/mautic.git mautic-product
cd mautic-product
git checkout 7.1        # branch (7.0, 7.1, 7.x, ...) or a release tag (7.1.3, 7.2.0-rc, ...)
```

- **Branches** (`7.0`, `7.1`, `7.x`) track the latest patch of a line.
- **Tags** (`git tag | grep '^7\.'`) pin an exact release for reproducible screenshots.
- This docs collection's default branch is **7.1**, so screenshots for these docs should
  come from `7.1` unless a page states otherwise.

Everything below runs from inside the `mautic-product` checkout.

---

## Path A — Native (no Docker)

### A1. Install the toolchain (macOS / Homebrew)

```bash
brew install php@8.2 mariadb composer symfony-cli/tap/symfony-cli
export PATH="/opt/homebrew/opt/php@8.2/bin:/opt/homebrew/opt/mariadb/bin:$PATH"
```

- PHP **8.2** matches Mautic's pinned `config.platform.php` (`8.2.0`). Newer PHP works
  too, but 8.2 is the safest match.
- `ext-imap` is **not** required for screenshots (it's for monitored inboxes). Native
  macOS PHP doesn't bundle it, so install skips it with a flag below.

### A2. Raise the CLI memory limit (required)

Symfony's container compile and Mautic's asset generation exceed PHP's default 128M and
fail with `Allowed memory size ... exhausted`. Set it unlimited for the CLI:

```bash
# drop a conf.d override (find your scan dir with: php --ini)
echo 'memory_limit = -1' > /opt/homebrew/etc/php/8.2/conf.d/zz-mautic-memory.ini
php -i | grep '^memory_limit'   # expect: memory_limit => -1 => -1
```

### A3. Start a workspace-local MariaDB (no root, no sudo)

Homebrew's shared MariaDB uses `unix_socket` auth for `root`, which needs OS root. A
sandbox has no sudo, so run a self-owned instance instead. macOS caps unix socket paths
at ~104 chars, so keep the socket in `/tmp` and connect over TCP.

```bash
DATADIR="$PWD/../mariadb-data"; SOCK="/tmp/mautic-mariadb.sock"
mariadb-install-db --no-defaults --datadir="$DATADIR" --auth-root-authentication-method=normal
mariadbd --no-defaults --datadir="$DATADIR" --socket="$SOCK" --port=3307 --bind-address=127.0.0.1 &

# wait for it, then create the database + app user
until mariadb -h 127.0.0.1 -P 3307 -u root -e 'SELECT 1' >/dev/null 2>&1; do sleep 1; done
mariadb -h 127.0.0.1 -P 3307 -u root <<'SQL'
CREATE DATABASE IF NOT EXISTS mautic CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER IF NOT EXISTS 'mautic'@'127.0.0.1' IDENTIFIED BY 'mautic';
GRANT ALL PRIVILEGES ON mautic.* TO 'mautic'@'127.0.0.1';
FLUSH PRIVILEGES;
SQL
```

### A4. Install PHP dependencies

```bash
composer install --no-interaction --no-progress --ignore-platform-req=ext-imap
```

### A5. Point Mautic at the database

Write `config/local.php` (overwrites any DDEV-generated one):

```php
<?php
$parameters = [
    'db_driver'      => 'pdo_mysql',
    'db_host'        => '127.0.0.1',
    'db_port'        => 3307,
    'db_name'        => 'mautic',
    'db_user'        => 'mautic',
    'db_password'    => 'mautic',
    'db_table_prefix'=> null,
    'admin_email'    => 'admin@localhost.test',
    'admin_password' => 'Maut1cR0cks!',
    'mailer_dsn'     => 'null://null',
    'install_source' => 'Native',
];
```

### A6. Keep `APP_ENV=prod` for clean screenshots

`.env` already sets `APP_ENV=prod`. Do **not** copy DDEV's `.ddev/.env.local.dist` to
`.env.local` — it sets `APP_ENV=dev`, which renders the Symfony debug toolbar over every
page and pollutes screenshots.

```bash
rm -f .env.local   # remove any dev override
```

### A7. Install Mautic (headless) and serve it

The install URL must match the URL you serve on.

```bash
php bin/console mautic:install "http://localhost:8001" --force --no-interaction
php bin/console cache:warmup --env=prod --no-interaction
php bin/console mautic:plugins:reload

symfony server:start --port=8001 --no-tls --dir="$PWD" &
```

> A `doctrine:migrations:version ... metadata storage is not initialized` warning during
> install is expected and non-fatal — the same warning appears under DDEV. The install
> still completes ("Install complete") and the schema (~123 tables) is created.

Verify:

```bash
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:8001/s/login   # expect 200
```

### A8. Log in

- URL: `http://localhost:8001`
- Username: `admin`
- Password: `Maut1cR0cks!`

---

## Path B — DDEV (Docker; human convenience only)

The `mautic-product` repo ships a `.ddev/` config whose `post-start` hook runs the full
install automatically. Requires Docker (or Colima) + DDEV installed.

```bash
brew install ddev/ddev/ddev mkcert   # one-time
cd mautic-product
git checkout 7.1
ddev start                           # builds containers, composer install, mautic:install
```

Serves at `https://<projectdir>.ddev.site` (e.g. `https://mautic-product.ddev.site`),
login `admin` / `Maut1cR0cks!`. PHPMyAdmin on `:8037`, MailHog on `:8026`. Stop with
`ddev stop`. Note: DDEV installs with `APP_ENV=dev`, so the debug toolbar shows — for
clean screenshots prefer Path A, or set prod mode inside the web container.

---

## Capturing screenshots

Point browser automation (Playwright) at the local URL, log in, navigate, capture.

- **Log in once**, then navigate to the target screen by URL (`/s/dashboard`,
  `/s/contacts`, `/s/campaigns`, ...).
- **Retina / 2x**: capture at device scale (`scale: "device"` in Playwright, or a CDP
  `deviceScaleFactor: 2` override) so images are crisp — a 1440-wide viewport yields a
  ~2880-wide PNG.
- **Clean chrome**: confirm `APP_ENV=prod` (A6) so no debug toolbar appears.
- **Match the docs' version**: the screenshot's Mautic version must equal the version the
  page documents (section 1). Re-run against a different checkout to reshoot another
  version.

---

## Gotchas (all hit and resolved during setup)

| Symptom | Cause | Fix |
| --- | --- | --- |
| `Allowed memory size ... exhausted` | PHP CLI default 128M too small for Symfony compile / asset gen | `memory_limit = -1` (A2); child scripts inherit `php.ini`, not `COMPOSER_MEMORY_LIMIT` |
| `ext-imap ... missing` on `composer install` | `mautic/core-lib` requires imap; not bundled in native macOS PHP; not needed for screenshots | `--ignore-platform-req=ext-imap` |
| `Can't create UNIX socket` | MariaDB socket path > ~104 chars (macOS limit) | short socket in `/tmp`, connect over TCP `127.0.0.1:3307` |
| `Access denied for 'root'@'localhost'` | Homebrew MariaDB `root` uses `unix_socket` (needs OS root/sudo) | own datadir with `--auth-root-authentication-method=normal` (A3) |
| Debug toolbar in screenshots | DDEV's `.env.local` sets `APP_ENV=dev` | `rm .env.local`; keep `APP_ENV=prod` (A6) |
| `doctrine:migrations:version` error mid-install | Known non-fatal Mautic install quirk | ignore — install still completes |

---

## Teardown

```bash
symfony server:stop
mariadb-admin -h 127.0.0.1 -P 3307 -u root shutdown   # or: kill the mariadbd process
# optionally: rm -rf ../mariadb-data
```
