# LibreWiki Homelab

A MediaWiki homelab environment for experimenting with **Libre Wiki compatibility, content migration, and Linux server administration**.

This project started from a simple idea: rather than building a new wiki from scratch, I wanted a laboratory where I could import existing Libre Wiki pages, study how the wiki is structured, reproduce its environment, and experiment with improvements without affecting the original service.

It also serves as a practical homelab project for learning and documenting Linux server administration.

> [!NOTE]
> This project is **not an official Libre Wiki service or official mirror**.
> Libre Wiki content itself is not distributed through this repository.

## Goals

The main goals of this project are:

- Build and operate a MediaWiki environment from scratch
- Reproduce parts of the Libre Wiki environment
- Test migration of MediaWiki pages and revision history
- Investigate dependencies between templates, CSS, gadgets, and extensions
- Experiment with wiki structure and categorization safely
- Practice Linux server administration using a real application
- Document troubleshooting and infrastructure changes
- Eventually separate services into multiple virtual machines

## Current Environment

### Host

- MacBook Air 13"
- Apple M5
- UTM

### Virtual Machine

- Ubuntu Server 26.04 LTS
- ARM64
- 4 GiB RAM
- 64 GiB virtual disk

### Application Stack

- Nginx
- PHP 8.5 / PHP-FPM
- MariaDB
- MediaWiki 1.46
- Liberty skin

Current architecture:

```text
macOS
└── UTM
    └── Ubuntu Server
        ├── Nginx
        ├── PHP-FPM
        ├── MediaWiki
        └── MariaDB
```

This intentionally starts as a simple single-server setup. Services will be separated gradually as the homelab develops.

## Why Libre Wiki?

My main interest in wikis is not necessarily writing complete articles from scratch.

I am more interested in:

- reorganizing categories
- improving existing document structures
- adding or improving individual sections
- connecting related pages
- experimenting with information architecture

A local Libre Wiki-compatible environment makes it possible to experiment with larger structural changes without disrupting the original wiki.

Instead of treating the fork as a replacement for Libre Wiki, the current goal is closer to a **staging and research environment**.

## Current Progress

### Infrastructure

- [x] Created an ARM64 Ubuntu Server VM with UTM
- [x] Configured SSH access
- [x] Configured SSH public-key authentication
- [x] Installed and configured Nginx
- [x] Installed PHP-FPM
- [x] Installed and secured MariaDB
- [x] Created a dedicated MediaWiki database and database user
- [x] Installed MediaWiki
- [x] Installed the Liberty skin

### Libre Wiki compatibility

- [x] Imported test pages from Libre Wiki
- [x] Added TemplateStyles support
- [x] Added ParserFunctions support
- [x] Added Cite support
- [x] Added Gadgets support
- [x] Imported required portions of `MediaWiki:Common.css`
- [x] Restored `hlist` rendering
- [x] Restored the Righteditlinks gadget
- [x] Adapted section edit-link styling for newer MediaWiki heading markup
- [ ] Configure VisualEditor
- [ ] Configure Scribunto and Lua modules
- [ ] Investigate additional Libre Wiki extensions
- [ ] Test more complex templates
- [ ] Investigate file/image migration

## Troubleshooting Notes

One of the main purposes of this repository is documenting problems encountered while reproducing an existing MediaWiki environment.

### 413 Request Entity Too Large

Large XML imports were initially rejected by Nginx.

The Nginx request body limit had to be increased:

```nginx
client_max_body_size 50M;
```

PHP upload limits also had to be adjusted accordingly.

Relevant PHP settings include:

```ini
upload_max_filesize
post_max_size
memory_limit
max_execution_time
max_input_time
```

### Session data lost during XML import

After increasing the Nginx upload limit, MediaWiki reported errors such as:

```text
Loss of session data
No interwiki prefix was supplied
```

The uploaded POST request was still exceeding PHP's request limits.

Increasing both the Nginx and PHP limits resolved the problem.

### `sanitized-css` content model not registered

Imported pages contained TemplateStyles CSS pages using the `sanitized-css` content model.

MediaWiki could not import them until the **TemplateStyles** extension was enabled.

### Parser functions not working

Templates containing parser functions such as:

```wiki
{{#if: ... }}
```

did not render correctly.

Enabling **ParserFunctions** restored them.

### References not rendering

Imported pages using:

```wiki
<ref>...</ref>
```

required the **Cite** extension.

### `hlist` layout was broken

Navigation templates using the `hlist` class appeared vertically instead of horizontally.

The relevant `hlist` CSS rules were defined in Libre Wiki's site-level CSS rather than the base Liberty skin.

Importing the required rules from:

```text
MediaWiki:Common.css
```

restored the expected horizontal layout.

### Righteditlinks and MediaWiki heading markup

Libre Wiki's Righteditlinks gadget was originally designed around an older MediaWiki heading structure.

Older structure:

```html
<h2>
  <span class="mw-headline">Heading</span>
  <span class="mw-editsection">Edit</span>
</h2>
```

The newer MediaWiki environment uses a wrapper:

```html
<div class="mw-heading mw-heading2">
  <h2>Heading</h2>
  <span class="mw-editsection">Edit</span>
</div>
```

As a result, the original gadget CSS placed section edit links incorrectly relative to Liberty's heading border.

The compatibility fix involved:

- adapting `.mw-heading` to the new structure
- removing the old floating behavior where necessary
- positioning `.mw-editsection` within the new heading wrapper
- moving the visual heading border from the inner `h2` to the wrapper

This was the first case where reproducing the original wiki required **porting an existing customization rather than simply copying it**.

## Planned Architecture

The current all-in-one VM is only the first stage.

A future version may look like this:

```text
MacBook / Homelab
│
├── Ubuntu Server
│   ├── Nginx
│   ├── PHP-FPM
│   └── MediaWiki
│
├── Rocky Linux
│   └── MariaDB
│
└── Monitoring
    ├── Uptime Kuma
    ├── Prometheus
    └── Grafana
```

Moving MariaDB to a Rocky Linux VM will provide an opportunity to practice:

- RHEL-family Linux administration
- `dnf`
- `firewalld`
- SELinux
- remote database configuration
- host-based database permissions
- network segmentation

## Future Plans

### Infrastructure

- [ ] Move MariaDB to a separate Rocky Linux VM
- [ ] Configure internal networking between application and database servers
- [ ] Add firewall rules
- [ ] Add automated database backups
- [ ] Test full disaster recovery
- [ ] Add monitoring
- [ ] Add service health checks
- [ ] Document backup and restoration procedures
- [ ] Experiment with Ansible
- [ ] Compare VM-based and container-based deployment

### MediaWiki

- [ ] Install additional extensions used by Libre Wiki
- [ ] Configure VisualEditor
- [ ] Configure Scribunto
- [ ] Reproduce additional gadgets
- [ ] Investigate templates and Lua dependencies
- [ ] Import a larger test dataset
- [ ] Develop a repeatable import workflow
- [ ] Investigate automated archival/synchronization

### Networking

Later stages of the homelab may introduce separate networks for application and database services.

For example:

```text
Application Network
192.168.10.0/24
        │
        │ Firewall / Routing
        ▼
Database Network
192.168.20.0/24
```

This will be used to practice routing, subnetting, firewall rules, and network access control.

## Repository Structure

Planned repository structure:

```text
librewiki-homelab/
├── README.md
├── docs/
│   ├── 01-vm-setup.md
│   ├── 02-ssh.md
│   ├── 03-nginx-php.md
│   ├── 04-mariadb.md
│   ├── 05-mediawiki.md
│   ├── 06-liberty.md
│   ├── 07-import.md
│   ├── 08-extensions.md
│   └── 09-troubleshooting.md
├── nginx/
│   └── librewiki.conf.example
├── mediawiki/
│   ├── LocalSettings.example.php
│   └── compatibility.css
├── scripts/
└── diagrams/
```

The structure will evolve as the project grows.

## Security

Sensitive production-style configuration is intentionally excluded from this repository.

The following should never be committed:

```text
LocalSettings.php
.env
*.sql
*.xml
*.key
*.pem
backup/
```

Configuration examples in this repository must use placeholder credentials.

For example:

```php
$wgDBname = 'librewiki';
$wgDBuser = 'librewiki';
$wgDBpassword = 'CHANGE_ME';
```

SSH private keys and actual database credentials are never stored in the repository.

## Content and Licensing

Libre Wiki pages may have their own licensing and attribution requirements.

This repository is primarily intended to contain:

- infrastructure configuration
- documentation
- compatibility fixes
- scripts written for this homelab
- diagrams and operational notes

Imported Libre Wiki database dumps, XML exports, and article content are **not included by default**.

When external code or styles are adapted, their original source and applicable license should be documented separately.

## Motivation

A homelab is useful when it has something meaningful to operate.

Rather than installing services only for the sake of practicing individual technologies, this project uses a real MediaWiki environment to connect several areas of infrastructure work:

```text
Linux
  ↓
Networking
  ↓
Web server
  ↓
PHP runtime
  ↓
Database
  ↓
MediaWiki
  ↓
Migration
  ↓
Troubleshooting
  ↓
Backup / Monitoring / Recovery
```

The goal is not simply to get MediaWiki running.

The goal is to understand how the entire service fits together, deliberately modify it, break parts of it, recover it, and document what was learned along the way.
