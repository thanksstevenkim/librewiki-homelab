# 03. Nginx and PHP-FPM

This document describes how Nginx and PHP-FPM were installed and connected on the LibreWiki Homelab Ubuntu Server VM.

The goal of this stage was to turn the VM into a functional web application server capable of serving both static files and PHP applications such as MediaWiki.

The final request path is:

```text
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
PHP application
```

## Environment

Guest system:

- Ubuntu Server 26.04 LTS
- ARM64
- UTM virtual machine

Application VM:

```text
librewiki-ubuntu
```

Software installed during this stage:

- Nginx
- PHP 8.5
- PHP-FPM
- MediaWiki-related PHP extensions

## Installing Nginx

The package index was refreshed first:

```bash
sudo apt update
```

Nginx was then installed:

```bash
sudo apt install nginx
```

Ubuntu automatically registers the Nginx service with systemd.

## Checking the Nginx Service

The service status can be inspected with:

```bash
systemctl status nginx
```

A healthy service should report:

```text
active (running)
```

A shorter check is:

```bash
systemctl is-active nginx
```

Expected output:

```text
active
```

## Testing Nginx from macOS

Because the VM was already reachable over the UTM virtual network, the Nginx default page could be accessed directly from the Mac.

For example:

```text
http://192.168.64.x
```

The actual VM IPv4 address should be substituted.

Successful access confirms that:

```text
Safari
   │
   │ TCP 80
   ▼
Ubuntu VM
   │
   ▼
Nginx
```

is working.

At this point the server was able to serve static HTTP content.

## Nginx Configuration Test

Before reloading or restarting Nginx after configuration changes, syntax should be checked with:

```bash
sudo nginx -t
```

A successful test returns output similar to:

```text
syntax is ok
test is successful
```

This became a standard workflow for later configuration changes:

```text
Edit configuration
        ↓
sudo nginx -t
        ↓
Syntax OK?
        ↓
sudo systemctl reload nginx
```

Testing before reload reduces the risk of taking the web server offline because of a configuration typo.

## Understanding the Default Web Root

Ubuntu's default Nginx site serves files from:

```text
/var/www/html
```

The default site configuration is located at:

```text
/etc/nginx/sites-available/default
```

and normally enabled through a symbolic link under:

```text
/etc/nginx/sites-enabled/
```

This default configuration was used temporarily to verify the basic web server before MediaWiki received its own site configuration.

## Installing PHP

MediaWiki requires PHP, while Nginx itself does not execute PHP directly.

PHP therefore needs a separate process manager.

The following packages were installed:

```bash
sudo apt install \
    php-fpm \
    php-cli \
    php-mysql \
    php-mbstring \
    php-xml \
    php-intl \
    php-curl \
    php-gd \
    php-zip
```

These provide:

- PHP command-line tools
- PHP-FPM
- MariaDB/MySQL connectivity
- multibyte text support
- XML processing
- internationalization
- HTTP client support
- image-processing support
- ZIP support

Additional extensions may be introduced later depending on MediaWiki extensions and imported content.

## Checking the PHP Version

The installed CLI version was checked with:

```bash
php -v
```

The installed environment reported:

```text
PHP 8.5.4 (cli)
```

This confirmed that PHP was installed correctly and available from the shell.

## PHP CLI vs PHP-FPM

It is important to distinguish between:

```text
php
```

and:

```text
php-fpm
```

The CLI binary is used when PHP scripts are executed manually from the terminal.

For example:

```bash
php script.php
```

Web requests use PHP-FPM instead.

The web request flow is:

```text
Nginx
   │
   │ FastCGI request
   ▼
PHP-FPM
   │
   ▼
PHP interpreter
```

This means changing CLI PHP configuration does not always change the configuration used by web requests.

For web troubleshooting, the FPM configuration is the important one.

## Checking PHP-FPM

The installed FPM service was:

```text
php8.5-fpm
```

Its status was checked with:

```bash
systemctl status php8.5-fpm
```

The service reported:

```text
active (running)
```

The service included one master process and multiple worker processes.

A healthy status included information similar to:

```text
Processes active: 0, idle: 2
```

## Checking the PHP-FPM Socket

Nginx communicates with PHP-FPM using a Unix socket in this setup.

Available PHP runtime files were inspected with:

```bash
ls /run/php
```

The relevant socket was:

```text
/run/php/php8.5-fpm.sock
```

This path later became part of the Nginx FastCGI configuration.

## Connecting Nginx to PHP-FPM

The default Nginx configuration was edited:

```bash
sudo nano /etc/nginx/sites-available/default
```

The `index` directive was updated to include PHP:

```nginx
index index.php index.html index.htm index.nginx-debian.html;
```

A PHP location block was then enabled:

```nginx
location ~ \.php$ {
    include snippets/fastcgi-php.conf;
    fastcgi_pass unix:/run/php/php8.5-fpm.sock;
}
```

The important part is:

```nginx
fastcgi_pass unix:/run/php/php8.5-fpm.sock;
```

This tells Nginx where to send PHP requests.

The resulting architecture is:

```text
HTTP request for .php
        │
        ▼
      Nginx
        │
        │ Unix socket
        ▼
/run/php/php8.5-fpm.sock
        │
        ▼
    PHP-FPM
```

## Applying the Nginx Configuration

After editing the configuration, syntax was tested:

```bash
sudo nginx -t
```

After a successful test, Nginx was reloaded:

```bash
sudo systemctl reload nginx
```

A reload is preferable to a full restart when only configuration has changed because existing connections can continue while the configuration is re-read.

## Testing PHP through Nginx

A temporary test file was created:

```bash
sudo nano /var/www/html/info.php
```

with:

```php
<?php
phpinfo();
```

The file was then accessed from macOS:

```text
http://192.168.64.x/info.php
```

A PHP information page confirmed that the full chain was working:

```text
Safari
   ↓
Nginx
   ↓
PHP-FPM
   ↓
PHP 8.5
   ↓
Generated HTML
```

This test verified more than simply running:

```bash
php -v
```

because it proved that PHP worked specifically through the web server and FPM environment.

## Removing the PHP Information Page

The `phpinfo()` output exposes a large amount of server information, including:

- PHP version
- extensions
- environment details
- filesystem paths
- request configuration
- server variables

It should therefore not remain accessible.

After confirming PHP operation, the test file was removed:

```bash
sudo rm /var/www/html/info.php
```

## Important PHP Configuration Paths

PHP maintains configuration separately for different execution modes.

The FPM configuration used by Nginx is located under:

```text
/etc/php/8.5/fpm/
```

The primary configuration file is:

```text
/etc/php/8.5/fpm/php.ini
```

The CLI configuration is separate:

```text
/etc/php/8.5/cli/php.ini
```

This distinction later became important when troubleshooting large MediaWiki XML imports.

Running:

```bash
php -i
```

reports the CLI PHP configuration, which may not be identical to the FPM configuration used by Nginx.

## Upload Limits

PHP imposes request and upload limits independently of Nginx.

Important settings include:

```ini
upload_max_filesize
post_max_size
memory_limit
max_execution_time
max_input_time
```

These are configured for web requests in:

```text
/etc/php/8.5/fpm/php.ini
```

For example:

```ini
upload_max_filesize = 64M
post_max_size = 80M
memory_limit = 256M
max_execution_time = 300
max_input_time = 300
```

These values were not initially necessary for the basic PHP setup, but later became important when importing large MediaWiki XML files.

After changing FPM configuration, PHP-FPM must be restarted:

```bash
sudo systemctl restart php8.5-fpm
```

## Nginx Request Size Limit

Nginx also independently limits request body size.

This later caused:

```text
413 Request Entity Too Large
```

during MediaWiki XML import.

The relevant Nginx directive is:

```nginx
client_max_body_size 50M;
```

It can be placed in the relevant `server` block.

For large uploads, all relevant limits must be considered together:

```text
Browser upload
     │
     ▼
Nginx client_max_body_size
     │
     ▼
PHP post_max_size
     │
     ▼
PHP upload_max_filesize
     │
     ▼
MediaWiki
```

If any layer has a smaller limit than the uploaded request, the operation can fail before MediaWiki itself processes the file.

## Why Nginx and PHP-FPM Are Separate

One important lesson from this stage is that the web server and application runtime have separate responsibilities.

Nginx handles:

- HTTP connections
- static files
- routing
- request limits
- reverse-proxy behavior
- FastCGI forwarding

PHP-FPM handles:

- PHP application execution
- PHP worker processes
- runtime limits
- PHP modules
- application code

This separation becomes useful during troubleshooting.

For example:

```text
Nginx 413 error
```

usually means the request was rejected before PHP processed it.

A PHP upload or execution error means the request passed through Nginx and failed at a later layer.

Understanding the layer where a failure occurs helps narrow down the cause.

## Service Management

Useful Nginx commands:

```bash
systemctl status nginx
sudo systemctl reload nginx
sudo systemctl restart nginx
sudo nginx -t
```

Useful PHP-FPM commands:

```bash
systemctl status php8.5-fpm
sudo systemctl restart php8.5-fpm
```

## Nginx Logs

Nginx logs are stored under:

```text
/var/log/nginx/
```

Important files include:

```text
/var/log/nginx/access.log
/var/log/nginx/error.log
```

The error log can be inspected with:

```bash
sudo tail -n 50 /var/log/nginx/error.log
```

or followed in real time:

```bash
sudo tail -f /var/log/nginx/error.log
```

This later became useful when debugging MediaWiki and PHP behavior.

## PHP-FPM Logs

PHP-FPM is managed by systemd, so its service logs can be inspected with:

```bash
journalctl -u php8.5-fpm
```

For recent errors:

```bash
sudo journalctl -u php8.5-fpm -n 50
```

For live output:

```bash
sudo journalctl -u php8.5-fpm -f
```

## Initial Architecture After This Stage

At the end of this stage, the VM contained:

```text
Ubuntu Server
│
├── Nginx
│     │
│     └── HTTP :80
│
└── PHP-FPM 8.5
      │
      └── /run/php/php8.5-fpm.sock
```

The request flow was:

```text
MacBook
   │
   │ HTTP
   ▼
Nginx
   │
   │ FastCGI
   ▼
PHP-FPM
```

MariaDB and MediaWiki had not yet been added to the architecture at the beginning of this stage.

## Security Considerations

The initial environment uses plain HTTP because it runs on a private UTM network and is not publicly exposed.

TLS/HTTPS is intentionally deferred until a later stage.

The PHP information page used during testing was deleted immediately after verification.

Configuration files containing secrets should never be committed to the public Git repository.

## Future Improvements

Possible future Nginx and PHP exercises include:

- configure HTTPS
- use a dedicated hostname
- configure reverse proxying
- add security headers
- tune PHP-FPM workers
- configure custom access and error logs
- investigate caching
- configure HTTP/2 or HTTP/3
- add rate limiting
- test application failure behavior
- move Nginx into a dedicated reverse-proxy role
- separate application and proxy servers
- compare manual deployment with containers

## Result

This stage is complete when:

- Nginx is installed and running
- the Nginx default page is reachable from macOS
- PHP 8.5 is installed
- PHP-FPM is running
- `/run/php/php8.5-fpm.sock` exists
- Nginx forwards PHP requests to PHP-FPM
- a test PHP page renders correctly through Nginx
- the temporary `phpinfo()` page is removed
- Nginx configuration can be safely tested and reloaded

The next stage is installing MariaDB and creating a dedicated database environment for MediaWiki.
