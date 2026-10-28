# Installing iHRIS 4.3.4 on Ubuntu


## Overview

This guide installs iHRIS 4.3.4 (Manage, Qualify or Train) on Ubuntu with **Apache 2.4**, **MySQL 8** and **PHP**. iHRIS 4.3.4 runs on **PHP 7.2 to 8.5**, so the steps work on any recent Ubuntu LTS release.

!!! note
    **The commands use PHP 8.3.** If you use another PHP version, replace `8.3` with it everywhere: `php8.4-gd` instead of `php8.3-gd`, `/etc/php/8.4/` instead of `/etc/php/8.3/`, and so on. `php -v` shows which version you have.

### Which PHP version to use

Use **PHP 8.3 or later**. Older versions run iHRIS 4.3.4 but are not recommended, because they no longer get security fixes. When a security flaw is found in an unsupported PHP version, it is published but never fixed, so anyone can use it to attack servers still running that version.

| PHP | Security fixes from the PHP project until | Use it? |
| --- | --- | --- |
| 7.2 to 8.1 | Already ended | No |
| 8.2 | 31 Dec 2026 | No, support ends soon |
| 8.3 | 31 Dec 2027 | Yes (used in this guide) |
| 8.4 | 31 Dec 2028 | Yes |
| 8.5 | 31 Dec 2029 | Yes |

Dates from [PHP supported versions](https://www.php.net/supported-versions.php). Ubuntu keeps patching the PHP version it ships for as long as that Ubuntu release is supported, but a newer PHP is still the safer choice.

| Ubuntu release | PHP it ships | MySQL it ships |
| --- | --- | --- |
| 22.04 LTS | 8.1 | 8.0 |
| 24.04 LTS | 8.3 | 8.0 |
| 26.04 LTS | 8.5 | 8.4 |

To use a PHP version your Ubuntu release doesn't ship (for example PHP 8.3 on 22.04), see Step 1.

**Tested:** iHRIS Manage, Qualify and Train 4.3.4 passed a full test run on PHP 7.2, 7.3, 7.4, 8.0, 8.1, 8.2, 8.3, 8.4 and 8.5, with MySQL 8.0 on Ubuntu 24.04. MySQL 8.4 (Ubuntu 26.04) has not been tested yet.

## Step 1: Install PHP and Apache

Installing PHP also installs Apache and the Apache PHP module (`libapache2-mod-php8.3`). Install the MySQL driver at the same time, since iHRIS connects to MySQL through PDO:

```bash
sudo apt update
sudo apt install php8.3 php8.3-mysql
```

If `apt` says it can't find the package, your Ubuntu release doesn't ship that PHP version. Add the [Ondřej Surý PHP repository](https://launchpad.net/~ondrej/+archive/ubuntu/php), which packages every PHP version from 5.6 to 8.5 for Ubuntu, then run the install again:

```bash
sudo apt install software-properties-common
sudo add-apt-repository ppa:ondrej/php
sudo apt update
sudo apt install php8.3 php8.3-mysql
```

Check the version:

```bash
php -v      # PHP 8.3.x
```

If more than one PHP version is installed, make sure Apache and the command line use the same one: `sudo update-alternatives --set php /usr/bin/php8.3`, and enable only that version's Apache module (`sudo a2dismod php8.1 && sudo a2enmod php8.3`).

**Optional: email.** iHRIS sends email (for example, password resets) through the server's mail system. If you need it, install Postfix:

```bash
sudo apt install postfix
sudo dpkg-reconfigure postfix
```

## Step 2: Install MySQL and set the root password

```bash
sudo apt install mysql-server
sudo mysql_secure_installation
```

On Ubuntu, MySQL's `root` user logs in through the system account (`auth_socket`), not a password. iHRIS and most PHP tools need a password, so set one:

```bash
sudo mysql
```

```sql
ALTER USER 'root'@'localhost' IDENTIFIED WITH caching_sha2_password BY 'YourStrongPassword';
FLUSH PRIVILEGES;
EXIT;
```

From now on, log in with `mysql -u root -p`. Plain `sudo mysql` no longer works.

!!! note
    **Password policy.** If `mysql_secure_installation` turned on password validation, a weak password is rejected with `ERROR 1819 … does not satisfy the current policy requirements`. Use a stronger password. On a local test machine only, you can remove the policy instead: `UNINSTALL COMPONENT 'file://component_validate_password';`

!!! note
    **PHP 7.2 or 7.3 only.** These versions can't log in with `caching_sha2_password`, MySQL's default; the site fails with `SQLSTATE[HY000] [2054] The server requested authentication method unknown to the client`. Use `IDENTIFIED WITH mysql_native_password` for the iHRIS database user in Step 7 instead. On MySQL 8.4 (Ubuntu 26.04) that method is turned off by default: add `mysql_native_password = ON` under `[mysqld]` in `/etc/mysql/mysql.conf.d/ihris.cnf` (Step 3). PHP 7.4 and later work with the default.

## Step 3: Configure MySQL for iHRIS

iHRIS needs a few MySQL settings. Set them in separate files under `/etc/mysql/mysql.conf.d/`, so MySQL upgrades don't overwrite them.

**1. Remove any query cache settings.** MySQL 8 has no query cache, and **MySQL will not start** if `query_cache_limit` or `query_cache_size` is set. If either appears in `/etc/mysql/mysql.conf.d/mysqld.cnf`, comment it out:

```bash
sudo sed -i 's/^query_cache_/# removed in MySQL 8.0: query_cache_/' /etc/mysql/mysql.conf.d/mysqld.cnf
```

**2. Set the sql-mode.** Create `/etc/mysql/mysql.conf.d/sql-mode.cnf`. Don't add `NO_AUTO_CREATE_USER`: MySQL 8 removed it and won't start with it.

```bash
sudo nano /etc/mysql/mysql.conf.d/sql-mode.cnf
```

```ini
[mysqld]
sql-mode = "ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION"
```

**3. Allow iHRIS to create database functions.** iHRIS creates MySQL functions while it installs. Put the setting in a config file, so it survives MySQL restarts:

```bash
sudo nano /etc/mysql/mysql.conf.d/ihris.cnf
```

```ini
[mysqld]
log_bin_trust_function_creators = 1
```

**4. Restart MySQL and check the settings:**

```bash
sudo systemctl restart mysql
mysql -u root -p -e "SELECT @@sql_mode, @@log_bin_trust_function_creators;"
```

Expected output: `ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION` and `1`.

## Step 4: PHP extensions, APCu, memcached and OPcache

**1. PHP extensions.** Use the versioned `php8.3-*` names for your PHP version. `php-pear` and `php-text-password` keep their generic names:

```bash
sudo apt install php-pear php8.3-gd php8.3-tidy php8.3-intl php8.3-bcmath php-text-password php8.3-mbstring php8.3-uuid php8.3-zip
```

**2. APCu** (in-memory cache):

```bash
sudo apt install php8.3-apcu
sudo nano /etc/php/8.3/mods-available/apcu.ini
```

```ini
extension=apcu.so
apc.enabled=1
apc.write_lock=1
apc.shm_size=100M
apc.slam_defense=0
apc.enable_cli=1
```

**3. Memcached.** Install it, then make sure the service is running and starts at boot:

```bash
sudo apt install php8.3-memcached memcached
sudo systemctl enable --now memcached
```

**4. OPcache.** Edit `/etc/php/8.3/mods-available/opcache.ini`:

```bash
sudo nano /etc/php/8.3/mods-available/opcache.ini
```

```ini
zend_extension=opcache.so
opcache.jit=off
opcache.memory_consumption=128M
opcache.interned_strings_buffer=8
opcache.max_accelerated_files=4000
opcache.revalidate_freq=60
opcache.enable_cli=1
```

For a development machine, use `opcache.revalidate_freq=2`, so code changes show up within 2 seconds.

**5. Timezone.** PHP uses UTC unless told otherwise, so printed reports, report timestamps and cache times don't match local time. Set your timezone in both `php.ini` files (Apache and command line):

```bash
sudo sed -i 's#^;\?date.timezone =.*#date.timezone = Africa/Dar_es_Salaam#' /etc/php/8.3/apache2/php.ini /etc/php/8.3/cli/php.ini
sudo systemctl restart apache2
php -r 'echo date_default_timezone_get(), PHP_EOL;'
```

The example uses `Africa/Dar_es_Salaam`; replace it with your zone from the [PHP timezone list](https://www.php.net/manual/en/timezones.php). The last command prints the zone PHP now uses.

## Step 5: Configure Apache

Enable the rewrite module:

```bash
sudo a2enmod rewrite
```

Allow `.htaccess` files under `/var/www`. Edit `/etc/apache2/apache2.conf` and change the `<Directory /var/www/>` block to:

```bash
sudo nano /etc/apache2/apache2.conf
```

```apache
<Directory /var/www/>
    Options Indexes FollowSymLinks MultiViews
    AllowOverride All
    Require all granted
</Directory>
```

Check the configuration, then restart Apache and memcached:

```bash
sudo apache2ctl configtest      # must print "Syntax OK"
sudo systemctl restart apache2
sudo systemctl restart memcached
```

The web root is `/var/www/html`. That's where the iHRIS installation creates its symlinks.

## Verification checklist

Run these before installing iHRIS. Each should give the expected result.

```bash
# PHP version and extensions
php -v | head -1
php -m | grep -iE '^(pdo_mysql|mysqli|apcu|memcached|gd|tidy|intl|bcmath|mbstring|uuid|xml|xsl|Zend OPcache)$' | sort -u

# PEAR package
pear list | grep Text_Password

# Apache modules
sudo apache2ctl -M | grep -E 'rewrite|php'

# MySQL settings
mysql -u root -p -e "SELECT @@sql_mode, @@log_bin_trust_function_creators;"

# Services running and enabled at boot
for s in apache2 mysql memcached; do echo "$s: $(systemctl is-active $s), $(systemctl is-enabled $s)"; done
```

| Check | Expected |
| --- | --- |
| PHP version | `PHP 8.3.x` (or the version you chose) |
| PHP extensions | All 13 listed: `pdo_mysql`, `mysqli`, `apcu`, `memcached`, `gd`, `tidy`, `intl`, `bcmath`, `mbstring`, `uuid`, `xml`, `xsl`, `Zend OPcache` |
| PEAR | `Text_Password` |
| Apache modules | `rewrite_module` and `php_module` |
| MySQL | `ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION` and `1` |
| Services | `apache2`, `mysql` and `memcached` all `active, enabled` |

## Step 6: Get the iHRIS 4.3.4 code

Get the code from Launchpad, either with Bazaar or as release tarballs. Both give exactly the same 4.3.4 code. Bazaar also gives you the full history and lets you update with one command.

Every site needs four parts: **I2CE** (the framework), **ihris-common**, **textlayout** (PDF output) and the product, **ihris-manage**, **ihris-qualify** or **ihris-train**. Use 4.3.4 for all four; don't mix 4.3.4 with 4.3.3 parts.

### Option A: Bazaar (recommended)

**1. Install Bazaar.** On current Ubuntu this installs Breezy, the maintained successor; the `bzr` command still works:

```bash
sudo apt install bzr
bzr version
```

**2. Choose a branch.**

| Branch | What you get | Use it for |
| --- | --- | --- |
| `4.3.4-release` | Exactly the 4.3.4 release | Production sites |
| `4.3-dev` (`lp:<project>`) | 4.3.4 plus any later fixes | Development |
| `4.3.3-release` | The 2018 release. **Does not run on PHP 8** | Only reproducing an old 4.3.3 site on PHP 7 |

**3. Download the 4.3.4 release.** Add the line for each product you need:

```bash
sudo mkdir -p /var/lib/iHRIS/lib/4.3.4
cd /var/lib/iHRIS/lib/4.3.4
sudo bzr branch lp:~intrahealth+informatics/i2ce/4.3.4-release I2CE
sudo bzr branch lp:~intrahealth+informatics/ihris-common/4.3.4-release ihris-common
sudo bzr branch lp:~intrahealth+informatics/textlayout/4.3.4-release textlayout
sudo bzr branch lp:~intrahealth+informatics/ihris-manage/4.3.4-release ihris-manage
sudo bzr branch lp:~intrahealth+informatics/ihris-qualify/4.3.4-release ihris-qualify
sudo bzr branch lp:~intrahealth+informatics/ihris-train/4.3.4-release ihris-train
```

For development, use `4.3-dev` instead: the same commands with `lp:i2ce I2CE`, `lp:ihris-common`, `lp:textlayout`, `lp:ihris-manage` and so on, in a folder such as `/var/lib/iHRIS/lib/4.3-dev`.

### Option B: release tarballs

Each product has a `-full` tarball that contains all four parts. Download the one for your product from its Launchpad page ([Manage](https://launchpad.net/ihris-manage/+download), [Qualify](https://launchpad.net/ihris-qualify/+download), [Train](https://launchpad.net/ihris-train/+download)), for example:

```bash
sudo mkdir -p /var/lib/iHRIS/lib/4.3.4
cd /var/lib/iHRIS/lib/4.3.4
sudo wget https://launchpad.net/ihris-manage/4.3.4/4.3.4/+download/ihris-manage-full-4.3.4.tar.bz2
sudo tar xjf ihris-manage-full-4.3.4.tar.bz2
sudo mv i2ce I2CE
```

The tarball unpacks I2CE as lowercase `i2ce`; the last command renames it so the paths in this guide work. To install a second product next to the first, download its plain tarball (for example `ihris-qualify-4.3.4.tar.bz2`) into the same folder.

### Then, for either option

**4. Point `4.3` at your code folder.** The site configuration in Step 7 uses `/var/lib/iHRIS/lib/4.3`, so link it to the folder that holds the code:

```bash
sudo ln -sfn /var/lib/iHRIS/lib/4.3.4 /var/lib/iHRIS/lib/4.3     # or .../4.3-dev
ls -l /var/lib/iHRIS/lib/
```

**5. Updating later** (Bazaar only):

```bash
cd /var/lib/iHRIS/lib/4.3.4
for d in I2CE ihris-common textlayout ihris-*; do (cd "$d" && sudo bzr pull); done
```

After updating, run the site update described in [Upgrading an existing site to 4.3.4](#upgrading-an-existing-site-to-434), below.

**For development**, give your own user ownership of the code, so you can edit it and run `bzr` without `sudo`:

```bash
sudo chown -R "$USER": /var/lib/iHRIS/lib
```

## Step 7: Install an iHRIS site (blank site)

The paths use the `/var/lib/iHRIS/lib/4.3` link from Step 6. The commands below install iHRIS Manage; for Qualify or Train, use that product's names from this table everywhere instead:

|  | Manage | Qualify | Train |
| --- | --- | --- | --- |
| Code folder | `ihris-manage` | `ihris-qualify` | `ihris-train` |
| Site folder | `/var/lib/iHRIS/sites/manage` | `/var/lib/iHRIS/sites/qualify` | `/var/lib/iHRIS/sites/train` |
| Database and its user | `ihris_manage` | `ihris_qualify` | `ihris_train` |
| Site file | `iHRIS-Manage-BLANK.xml` | `iHRIS-Qualify-BLANK.xml` | `iHRIS-Train-BLANK.xml` |
| Address | `http://<server-ip>/manage` | `http://<server-ip>/qualify` | `http://<server-ip>/train` |

Several products can run on one server, each with its own site folder and database, sharing the code in `/var/lib/iHRIS/lib/4.3`.

**1. Create the database and its user.** Choose a password for the `ihris_manage` user; this guide calls it **DBPASS**. Avoid `@`, `:`, `/`, `#` and `?` in it, because it goes inside a connection address in step 4. MySQL 8 needs the user created before privileges are granted:

```bash
mysql -u root -p
```

```sql
CREATE DATABASE ihris_manage;
CREATE USER 'ihris_manage'@'localhost' IDENTIFIED BY 'DBPASS';
GRANT ALL PRIVILEGES ON ihris_manage.* TO 'ihris_manage'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

On PHP 7.2 or 7.3, write `IDENTIFIED WITH mysql_native_password BY 'DBPASS'` in the `CREATE USER` line instead (see the note in Step 2).

iHRIS 4.3 creates MySQL functions while installing; that's why Step 3 set `log_bin_trust_function_creators = 1`.

**2. Copy the blank site:**

```bash
sudo mkdir -p /var/lib/iHRIS/sites
sudo cp -R /var/lib/iHRIS/lib/4.3/ihris-manage/sites/blank /var/lib/iHRIS/sites/manage
```

**3. (Optional) Set the feedback email address.** In `/var/lib/iHRIS/sites/manage/iHRIS-Manage-BLANK.xml`, find the `email` configuration and replace `root@localhost` with your address:

```bash
sudo nano /var/lib/iHRIS/sites/manage/iHRIS-Manage-BLANK.xml
```

```xml
<configuration name="email" path="to" values="single">
  <displayName>Email Address</displayName>
  <value>you@example.org</value>
</configuration>
```

**4. Configure the site.** Edit `config.values.php`:

```bash
sudo nano /var/lib/iHRIS/sites/manage/pages/config.values.php
```

Set these three lines. Remove the `//` in front of the last two:

```php
$i2ce_site_i2ce_path = "/var/lib/iHRIS/lib/4.3/I2CE";
$i2ce_site_dsn = 'mysql://ihris_manage:DBPASS@localhost/ihris_manage';
$i2ce_site_module_config = "/var/lib/iHRIS/sites/manage/iHRIS-Manage-BLANK.xml";
```

Use the full path rather than the template's default (`../../../../I2CE`), which assumes a different folder layout. Linux paths are case-sensitive, so the folder must be named `I2CE` exactly, as created in Step 6.

**5. Publish the site through Apache:**

```bash
sudo ln -s /var/lib/iHRIS/sites/manage/pages /var/www/html/manage
```

**6. (Optional) Clean URLs** (removes `index.php` from addresses):

```bash
sudo cp /var/www/html/manage/htaccess.TEMPLATE /var/www/html/manage/.htaccess
sudo nano /var/www/html/manage/.htaccess
```

Change `RewriteBase  /iHRIS/manage` to:

```apache
RewriteBase /manage
```

**7. Run the installer.** Open `http://<server-ip>/manage` in a browser and wait while the site sets itself up. The first load builds the database and can take several minutes. Then log in:

- **Username:** `i2ce_admin`
- **Password:** DBPASS (the database user's password)

**(Optional) phpMyAdmin**, for managing the database in a browser:

```bash
sudo apt install phpmyadmin     # choose apache2 with the spacebar when asked
```

Then open `http://<server-ip>/phpmyadmin`. Don't install it on a server reachable from the internet unless access to it is restricted.

## Upgrading an existing site to 4.3.4

This works for any 4.3 site (4.3.0 to 4.3.3, or `4.3-dev`). The site's database is updated in place and keeps its data. The examples use Manage; use your product's names from the table in Step 7.

**1. Back up the database and the site folder:**

```bash
mysqldump -u root -p --routines ihris_manage > ihris_manage-before-4.3.4.sql
sudo tar czf manage-site-before-4.3.4.tar.gz -C /var/lib/iHRIS/sites manage
```

**2. Install the 4.3.4 code** in a new folder, as in Step 6, and point the `4.3` link at it. If the site's `config.values.php` points `$i2ce_site_i2ce_path` somewhere else, change it to `/var/lib/iHRIS/lib/4.3/I2CE`.

**3. Update the site**, from its `pages` folder, as the web server user:

```bash
cd /var/lib/iHRIS/sites/manage/pages
sudo -u www-data php index.php --update=1
```

It takes a few minutes and ends with `Site was succesully updated` (spelled that way by iHRIS).

**4. Restart memcached and Apache:**

```bash
sudo systemctl restart memcached apache2
```

**5. Check the site.** Log in, open a person record and run a report. Then restart memcached once more and reload the page: the site should open normally and not ask to be installed.

To go back to the old version, point the `4.3` link back to the old code folder and restore the database from the dump.

## Running iHRIS in production

**Watch the Apache error log.** iHRIS logs every PHP warning and deprecation notice to `/var/log/apache2/error.log`, with a stack trace, whatever `php.ini` says. On PHP 8.1 and later that's a lot of lines: on the test server the log reached 10.8 GB and filled the disk. Ubuntu rotates and compresses the Apache logs every day and keeps 14 days (`/etc/logrotate.d/apache2`); keep that in place, and check free space now and then:

```bash
df -h /
sudo du -sh /var/log/apache2
```

**If the disk does fill up**, free space first (for example `sudo truncate -s 0 /var/log/apache2/error.log`), restart MySQL, Apache and memcached, then check the iHRIS tables, which MySQL may have marked as crashed:

```bash
mysqlcheck -u root -p --auto-repair ihris_manage
```

**Keep errors off the screen.** In `/etc/php/8.3/apache2/php.ini`, `display_errors` must be `Off`, as it is by default on Ubuntu, so visitors never see file paths or error details.

**Back up the database** regularly with `mysqldump --routines`, as in the upgrade steps above.

## Troubleshooting

Common errors and how to fix them.

| Symptom | Cause | Fix |
| --- | --- | --- |
| MySQL won't start; log shows `Error while setting value '…NO_AUTO_CREATE_USER…' to 'sql_mode'` | `NO_AUTO_CREATE_USER` was removed in MySQL 8.0 | Remove it from `sql-mode.cnf` (Step 3) |
| MySQL won't start; log mentions `query_cache_size` or `query_cache_limit` | The query cache was removed in MySQL 8.0 | Comment those lines out in `mysqld.cnf` (Step 3) |
| `ERROR 1819: Your password does not satisfy the current policy requirements` | The password-strength policy from `mysql_secure_installation` | Use a stronger password (Step 2) |
| `ERROR 1698: Access denied for user 'root'@'localhost'` | `root` still uses `auth_socket` | Set a password with `sudo mysql` (Step 2) |
| `ERROR 1064 … near 'identified by'` when granting privileges | MySQL 8.0 removed `GRANT … IDENTIFIED BY` | `CREATE USER` first, then `GRANT` (Step 7) |
| iHRIS can't create database functions or triggers | `log_bin_trust_function_creators` is 0, e.g. after a restart | Add it to `ihris.cnf` (Step 3) |
| Site can't find I2CE, or shows a blank page | The I2CE folder is lowercase `i2ce` (tarball, or `bzr branch` without a folder name), or `$i2ce_site_i2ce_path` is wrong | Name the folder `I2CE` (Step 6) and use `/var/lib/iHRIS/lib/4.3/I2CE` (Step 7) |
| `Call to undefined function get_magic_quotes_gpc()` | iHRIS 4.3.3 or older code on PHP 8 | Use iHRIS 4.3.4 (Step 6, or Upgrading an existing site) |
| `could not find driver` or no `pdo_mysql` in `php -m` | The PHP MySQL driver isn't installed | `sudo apt install php8.3-mysql`, then restart Apache |
| memcached is `inactive` | Installed but not started | `sudo systemctl enable --now memcached` |
| `SQLSTATE[HY000] [2054] The server requested authentication method unknown to the client` | PHP 7.2 or 7.3 can't use MySQL's default caching_sha2_password | Create the database user with mysql_native_password (Step 2 note), or use PHP 8.3 or later |
| Disk full; MySQL tables marked as crashed | The Apache error log grew too large | See Running iHRIS in production |

To see why MySQL fails to start:

```bash
sudo journalctl -u mysql -n 30 --no-pager
sudo tail -30 /var/log/mysql/error.log
```

After fixing a MySQL config error, clear systemd's failed state before starting again:

```bash
sudo systemctl reset-failed mysql
sudo systemctl start mysql
```

## Coming from the Ubuntu 18.04 guide?

If you set up a server with the earlier [Ubuntu 18.04 supporting software guide](https://ihris-documentation.readthedocs.io/en/latest/docs/installation/linux-ubuntu-installation-supporting-software-18-04.html), these are the settings that differ:

| Area | 18.04 guide | This guide |
| --- | --- | --- |
| PHP | 7.2, generic `php-*` packages | 8.3 (or your version), versioned `php8.3-*` packages |
| PHP–MySQL driver | Installed by `tasksel lamp-server` | Installed explicitly: `php8.3-mysql` |
| MySQL passwords | `mysql_native_password` | MySQL's default `caching_sha2_password` (PHP 7.2 and 7.3 need `mysql_native_password`) |
| MySQL query cache | `query_cache_limit`, `query_cache_size` | Not set (removed in MySQL 8) |
| sql-mode | Includes `NO_AUTO_CREATE_USER` | Without it (removed in MySQL 8) |
| `log_bin_trust_function_creators` | `SET GLOBAL` (lost on restart) | Set in a config file |
| OPcache | Includes `opcache.fast_shutdown` | Without it (removed in PHP 7.2) |
| `4.3` code link | Points to a `4.3.0` folder that is never created | Points to the folder that holds the code |
| Database user | `GRANT … IDENTIFIED BY` | `CREATE USER`, then `GRANT` |
