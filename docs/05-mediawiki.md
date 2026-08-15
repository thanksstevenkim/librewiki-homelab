# 05. MediaWiki

This document describes how MediaWiki was installed and connected to the existing Nginx, PHP-FPM, and MariaDB stack in the LibreWiki Homelab.

The goal of this stage was to move from a generic PHP server to a functioning wiki application.

The architecture after this stage became:

```text id="8mnc0q"
Browser
   │
   │ HTTP
   ▼
Nginx
   │
   │ FastCGI
   ▼
PHP-FPM
   │
   ▼
MediaWiki
   │
   ▼
MariaDB
```

## Environment

Guest system:

- Ubuntu Server 26.04 LTS
- ARM64
- UTM virtual machine

Application stack:

- Nginx
- PHP 8.5
- PHP-FPM 8.5
- MariaDB
- MediaWiki 1.46

Application VM:

```text id="z82q42"
librewiki-ubuntu
```

The MariaDB database and database user were already created in the previous stage.

## Database Prepared Earlier

The following database configuration was prepared before installing MediaWiki:

```text id="s18v4h"
Database name: librewiki
Database user: librewiki
Database host: localhost
```

The database user has privileges only on:

```text id="9o6uxn"
librewiki.*
```

The database root account is not used by MediaWiki.

## Downloading MediaWiki

MediaWiki was downloaded from the official release server.

The download was performed from the Ubuntu VM:

```bash id="j0qo0k"
cd ~
wget https://releases.wikimedia.org/mediawiki/1.46/mediawiki-1.46.0.tar.gz
```

The archive was then extracted:

```bash id="i8x8pd"
tar -xzf mediawiki-1.46.0.tar.gz
```

The extracted directory was moved into:

```text id="6jknw9"
/var/www/mediawiki
```

with:

```bash id="u6mj8n"
sudo mv mediawiki-1.46.0 /var/www/mediawiki
```

## File Ownership

The MediaWiki directory was assigned to the same system user used by Nginx and PHP-FPM:

```bash id="ph34v7"
sudo chown -R www-data:www-data /var/www/mediawiki
```

The resulting application root is:

```text id="l32n36"
/var/www/mediawiki
```

This allows the web application to access the files it needs through PHP-FPM.

## Creating a Dedicated Nginx Site

Instead of continuing to use Ubuntu's default Nginx web root, a dedicated site configuration was created for MediaWiki.

The configuration file was created at:

```text id="768d9s"
/etc/nginx/sites-available/librewiki
```

using:

```bash id="co2txa"
sudo nano /etc/nginx/sites-available/librewiki
```

The initial configuration was:

```nginx id="ytoizk"
server {
    listen 80;
    server_name _;

    root /var/www/mediawiki;
    index index.php;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.5-fpm.sock;
    }

    location ~ /\.ht {
        deny all;
    }
}
```

## Nginx Configuration Explanation

### `listen 80`

```nginx id="xzc0ze"
listen 80;
```

Nginx accepts normal HTTP requests on TCP port 80.

HTTPS is intentionally not configured yet because the initial environment runs only inside the local UTM network.

### `server_name _`

```nginx id="tz53w1"
server_name _;
```

No production domain is used at this stage.

The wiki is accessed directly by its VM IP address.

### `root`

```nginx id="y7giyu"
root /var/www/mediawiki;
```

This makes the MediaWiki installation directory the document root for this site.

### `index`

```nginx id="k8hw2e"
index index.php;
```

MediaWiki requests should resolve through the PHP entry point.

### Main Location

```nginx id="9gsgrh"
location / {
    try_files $uri $uri/ /index.php?$query_string;
}
```

Nginx first checks whether the requested file or directory exists.

If it does not, the request is passed to MediaWiki's:

```text id="aqnwdr"
index.php
```

entry point.

### PHP-FPM

```nginx id="szbkli"
location ~ \.php$ {
    include snippets/fastcgi-php.conf;
    fastcgi_pass unix:/run/php/php8.5-fpm.sock;
}
```

PHP files are passed to PHP-FPM through the Unix socket:

```text id="dq689s"
/run/php/php8.5-fpm.sock
```

### `.ht` Files

```nginx id="z2l2o3"
location ~ /\.ht {
    deny all;
}
```

Requests for `.ht*` files are denied.

Nginx does not use Apache `.htaccess` files, so there is no reason to expose such files through the web server.

## Enabling the Site

The MediaWiki site configuration was enabled by creating a symbolic link:

```bash id="spmq6g"
sudo ln -s /etc/nginx/sites-available/librewiki /etc/nginx/sites-enabled/librewiki
```

The original default site was disabled:

```bash id="wfku18"
sudo rm /etc/nginx/sites-enabled/default
```

This leaves the MediaWiki site as the active default HTTP service.

## Testing Nginx

Before applying the new configuration:

```bash id="9yt80h"
sudo nginx -t
```

A successful test should report:

```text id="b8tn9r"
syntax is ok
test is successful
```

The configuration was then reloaded:

```bash id="72mmsn"
sudo systemctl reload nginx
```

## Opening MediaWiki for the First Time

MediaWiki was accessed from the Mac browser using the Ubuntu VM IPv4 address:

```text id="zx0i16"
http://192.168.64.x
```

Because no MediaWiki configuration existed yet, the application displayed the initial setup page.

The installer provided a link to begin configuring the wiki.

## Web Installer

The MediaWiki web installer was used to create the initial configuration.

The database settings entered were:

```text id="pd4y8q"
Database type: MariaDB / MySQL
Database host: localhost
Database name: librewiki
Database username: librewiki
Database password: <secret>
```

The password is intentionally not recorded in this repository.

## Wiki Identity

The wiki was configured as a local laboratory rather than an official Libre Wiki deployment.

A suitable lab-style name was used to distinguish the environment from the original service.

This is important because the project is intended for:

- migration experiments
- extension testing
- CSS compatibility work
- structural experimentation
- infrastructure exercises

It is not intended to impersonate the original service.

## Administrator Account

The installer also created the initial MediaWiki administrator account.

This account is used for:

- installing site configuration pages
- editing `MediaWiki:` namespace pages
- configuring gadgets
- importing XML
- managing wiki-level settings during experimentation

Administrative credentials are not stored in Git.

## Generating `LocalSettings.php`

At the end of the installer, MediaWiki generated:

```text id="2t17jk"
LocalSettings.php
```

The browser downloaded this file to the Mac.

This file contains essential MediaWiki configuration, including database credentials.

It is therefore sensitive.

## Transferring `LocalSettings.php`

The generated file was transferred from macOS to the Ubuntu VM using SCP.

From the Mac terminal:

```bash id="jvs78b"
scp ~/Downloads/LocalSettings.php steven@192.168.64.x:/tmp/
```

The actual VM IP should be substituted.

The file was then moved into the MediaWiki application directory:

```bash id="qqn6wq"
sudo mv /tmp/LocalSettings.php /var/www/mediawiki/
```

## Ownership and Permissions

Ownership was set to:

```bash id="orh3u7"
sudo chown www-data:www-data /var/www/mediawiki/LocalSettings.php
```

Permissions were restricted:

```bash id="cf6x09"
sudo chmod 640 /var/www/mediawiki/LocalSettings.php
```

This is important because the file contains information such as:

```php id="huu1zx"
$wgDBname
$wgDBuser
$wgDBpassword
```

The configuration should not be world-readable.

## Verifying the Installation

After placing:

```text id="p7bunn"
/var/www/mediawiki/LocalSettings.php
```

the wiki was reloaded in the browser.

The setup screen disappeared and the actual MediaWiki main page loaded successfully.

This confirmed that the complete application path was working:

```text id="q8puh1"
Browser
   ↓
Nginx
   ↓
PHP-FPM
   ↓
MediaWiki
   ↓
MariaDB
```

## Checking the Database

After successful installation, the MediaWiki database can be inspected:

```bash id="psxvw4"
sudo mariadb
```

Then:

```sql id="kzlt8z"
USE librewiki;
SHOW TABLES;
```

A successful MediaWiki installation creates many application tables.

This provides a useful verification that the installer successfully initialized the database schema.

## `LocalSettings.php`

The primary MediaWiki configuration file is:

```text id="wba27e"
/var/www/mediawiki/LocalSettings.php
```

Future wiki-level configuration is added here, including:

- skins
- extensions
- database settings
- upload settings
- site features
- debug configuration
- permissions
- namespaces
- cache configuration

For example, extensions are typically enabled with:

```php id="8yi4t8"
wfLoadExtension( 'ExtensionName' );
```

Skins are loaded with:

```php id="nprjqw"
wfLoadSkin( 'SkinName' );
```

## Git Security

The actual `LocalSettings.php` file must never be committed to the public repository.

The repository should exclude:

```gitignore id="nlwxtl"
LocalSettings.php
```

A public example configuration may instead contain placeholders:

```php id="769s4e"
$wgDBserver = 'localhost';
$wgDBname = 'librewiki';
$wgDBuser = 'librewiki';
$wgDBpassword = 'CHANGE_ME';
```

The example file can be named:

```text id="h4fkm9"
LocalSettings.example.php
```

## MediaWiki Maintenance Scripts

MediaWiki provides command-line maintenance scripts.

In the current version, they can be invoked through:

```bash id="4m1nse"
cd /var/www/mediawiki
sudo -u www-data php maintenance/run.php
```

For example, after installing certain extensions or changing database-related functionality:

```bash id="3hjib9"
sudo -u www-data php maintenance/run.php update
```

This updates the database schema where required.

The use of:

```text id="vrrctf"
sudo -u www-data
```

runs the maintenance script under the web service user rather than the interactive administrator account.

## MediaWiki Logs and Troubleshooting

A PHP or MediaWiki failure may appear indirectly through Nginx.

The Nginx error log can be checked with:

```bash id="ydivqf"
sudo tail -n 50 /var/log/nginx/error.log
```

Live monitoring:

```bash id="8hy27o"
sudo tail -f /var/log/nginx/error.log
```

PHP-FPM logs can be checked with:

```bash id="mhe4zl"
sudo journalctl -u php8.5-fpm -n 50
```

These became useful later when experimenting with skins, extensions, and imported content.

## MediaWiki Special Pages

Several MediaWiki special pages became useful for this project.

### `Special:Version`

Used to inspect:

- MediaWiki version
- installed extensions
- installed skins
- library versions

### `Special:Import`

Used later to import XML exported from Libre Wiki.

### `Special:Export`

Useful for generating MediaWiki XML exports.

### `Special:RecentChanges`

Useful for inspecting imported revisions and local changes.

### `Special:Preferences`

Used later for configuring user-level gadgets and editing preferences.

## Import Planning

The first goal after installing MediaWiki was not to immediately mirror all of Libre Wiki.

Instead, the plan was:

```text id="4z30w5"
Import a few pages
       ↓
Observe what breaks
       ↓
Identify missing dependency
       ↓
Install extension / CSS / gadget
       ↓
Import again
```

This incremental strategy became useful for understanding the original wiki's dependency structure.

Examples of dependencies discovered later include:

- TemplateStyles
- ParserFunctions
- Cite
- Gadgets
- site-level CSS
- compatibility fixes for newer MediaWiki heading markup

## File Upload and Import Limits

The initial MediaWiki installation worked correctly for normal page access, but larger XML imports later exposed request-size limits in both Nginx and PHP.

The relevant Nginx setting is:

```nginx id="82bqfe"
client_max_body_size
```

Relevant PHP-FPM settings include:

```ini id="mgohl8"
upload_max_filesize
post_max_size
memory_limit
max_execution_time
max_input_time
```

These limits are documented in more detail in the import and troubleshooting stages.

The important architectural lesson is that an upload may pass through multiple layers:

```text id="qlayap"
Browser
  ↓
Nginx
  ↓
PHP-FPM
  ↓
MediaWiki
```

Each layer can independently reject the request.

## Application Directory Layout

The relevant directory structure after installation is approximately:

```text id="300lyz"
/var/www/mediawiki/
├── index.php
├── LocalSettings.php
├── includes/
├── maintenance/
├── skins/
├── extensions/
├── resources/
├── languages/
└── ...
```

Important directories for later work include:

```text id="ek68ig"
skins/
extensions/
```

These are where the Liberty skin and additional MediaWiki extensions are installed.

## Current Nginx Layout

The site configuration is:

```text id="svq8lo"
/etc/nginx/sites-available/librewiki
```

and enabled through:

```text id="5gjhxv"
/etc/nginx/sites-enabled/librewiki
```

This separation follows the Debian/Ubuntu Nginx convention.

## Current Architecture

At the end of this stage:

```text id="8ruwa9"
MacBook Air M5
│
└── UTM
    │
    └── Ubuntu Server
        │
        ├── Nginx
        │     └── HTTP :80
        │
        ├── PHP-FPM 8.5
        │     └── /run/php/php8.5-fpm.sock
        │
        ├── MediaWiki 1.46
        │     └── /var/www/mediawiki
        │
        └── MariaDB
              └── librewiki database
```

At this point, the homelab has become a complete working web application environment rather than a collection of individually installed packages.

## Future Improvements

Planned MediaWiki work includes:

- install the Liberty skin
- import Libre Wiki test pages
- install required extensions
- reproduce selected site CSS
- configure Gadgets
- configure VisualEditor
- configure Scribunto
- test complex templates
- investigate file migration
- configure automated backups
- experiment with larger XML imports
- separate MariaDB onto a Rocky Linux VM
- add monitoring and health checks

## Result

This stage is complete when:

- MediaWiki files are installed under `/var/www/mediawiki`
- Nginx serves the MediaWiki application
- PHP requests are handled through PHP-FPM
- MediaWiki connects successfully to MariaDB
- the database schema is initialized
- `LocalSettings.php` is installed with restricted permissions
- the wiki main page loads successfully
- the real configuration file is excluded from Git

The next stage focuses on installing the Liberty skin and making the lab environment visually and structurally closer to Libre Wiki.
