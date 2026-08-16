# 09. Troubleshooting

This document summarizes the main problems encountered while building the LibreWiki Homelab and explains how each issue was diagnosed and resolved.

The purpose is not only to record fixes, but to preserve the reasoning pattern used during troubleshooting.

A recurring theme throughout this project has been:

> Identify which layer is failing before changing configuration.

The stack contains several independent layers:

```text id="1dt8kj"
Browser
   ↓
UTM Network
   ↓
Nginx
   ↓
PHP-FPM
   ↓
MediaWiki
   ↓
Extensions / Gadgets / CSS
   ↓
MariaDB
```

A visible error in the browser may originate from any of them.

## 1. VM Booted into UEFI Shell

### Symptom

After creating the UTM virtual machine, the VM booted into a UEFI shell instead of the Ubuntu installer.

The screen showed a prompt similar to:

```text id="1muqeu"
Shell>
```

### Cause

The selected installation file was not an Ubuntu ISO.

A `.torrent` file had been selected instead.

A torrent file only contains metadata for downloading another file and is not itself a bootable operating system image.

### Resolution

Download the actual ARM64 Ubuntu Server ISO.

The expected file should look similar to:

```text id="ubub11"
ubuntu-26.04-live-server-arm64.iso
```

Then attach the `.iso` file as the VM boot image in UTM.

### Lesson

Always verify the actual file type before diagnosing boot problems.

---

## 2. Ubuntu Installer Started Again After Installation

### Symptom

Ubuntu Server installation completed successfully, but restarting the VM launched the installer again.

### Cause

The installation ISO was still attached to the VM.

The virtual machine therefore continued booting from the installation media instead of the installed virtual disk.

### Resolution

Stop the VM and detach the Ubuntu ISO from the virtual CD/DVD drive in UTM.

Then boot the VM again.

### Lesson

The message:

```text id="e0nx51"
Remove the installation medium, then press Enter
```

has a direct virtual-machine equivalent.

In UTM, the installation medium is the attached ISO.

---

## 3. SSH Key Created on the Wrong Machine

### Symptom

An SSH key was accidentally created inside the Ubuntu VM instead of on the Mac.

### Cause

The `ssh-keygen` command was run from the server shell rather than the macOS terminal.

### Impact

No damage occurred.

The resulting key simply belongs to the Ubuntu VM.

### Resolution

Use the existing key on macOS for:

```text id="bt8l53"
Mac → Ubuntu
```

authentication.

The accidentally created Ubuntu key can remain available for future use, for example:

```text id="4cogee"
Ubuntu → Rocky Linux
```

when a second VM is introduced.

### Lesson

SSH keys belong to the machine initiating the connection.

---

## 4. Nginx Worked, but PHP Did Not

### Symptom

The Nginx default page loaded successfully, but PHP files were not executed.

### Cause

Nginx does not execute PHP directly.

PHP-FPM must be installed and configured as a FastCGI backend.

### Resolution

Install PHP-FPM and required modules.

Verify the socket:

```bash id="p2gknv"
ls /run/php
```

The relevant socket was:

```text id="dvza4u"
/run/php/php8.5-fpm.sock
```

Then configure Nginx:

```nginx id="6e5w0e"
location ~ \.php$ {
    include snippets/fastcgi-php.conf;
    fastcgi_pass unix:/run/php/php8.5-fpm.sock;
}
```

### Verification

Create a temporary:

```php id="hp6usa"
<?php
phpinfo();
```

page and load it through the browser.

### Lesson

A web application stack should be thought of as independent services rather than one combined application.

---

## 5. `413 Request Entity Too Large`

### Symptom

Importing a Libre Wiki XML export produced:

```text id="83v0fp"
413 Request Entity Too Large
```

### Layer

Nginx.

### Cause

The uploaded XML exceeded Nginx's configured request-body size.

### Resolution

Edit:

```text id="tm40id"
/etc/nginx/sites-available/librewiki
```

and add an appropriate request limit:

```nginx id="1hlslt"
client_max_body_size 50M;
```

Then validate the configuration:

```bash id="tbx7ot"
sudo nginx -t
```

and reload:

```bash id="jj61he"
sudo systemctl reload nginx
```

### Lesson

An Nginx-generated HTTP error means the request may not have reached PHP or MediaWiki at all.

Diagnose the outermost failing layer first.

---

## 6. Session Data Lost During XML Import

### Symptom

After fixing the Nginx limit, XML import produced errors such as:

```text id="hs9v7q"
Session data was lost.
```

and:

```text id="jdj5li"
No interwiki prefix was supplied.
```

The interwiki prefix field had already been filled.

### Layer

PHP-FPM request processing.

### Cause

The request had become large enough to pass Nginx but still exceeded PHP request limits.

When PHP discards an oversized POST request, MediaWiki may receive neither:

- uploaded file data
- form parameters
- CSRF/session information
- interwiki prefix

This makes the displayed MediaWiki errors misleading.

### Resolution

Inspect FPM values:

```bash id="ve16cu"
grep -E '^(upload_max_filesize|post_max_size|max_input_time|max_execution_time|memory_limit)' /etc/php/8.5/fpm/php.ini
```

Increase them as necessary.

For example:

```ini id="8rt8qr"
upload_max_filesize = 64M
post_max_size = 80M
memory_limit = 256M
max_execution_time = 300
max_input_time = 300
```

Restart PHP-FPM:

```bash id="qh2rm3"
sudo systemctl restart php8.5-fpm
```

### Lesson

A request may pass through several size limits:

```text id="upfnt3"
Nginx
   ↓
PHP post_max_size
   ↓
PHP upload_max_filesize
   ↓
MediaWiki
```

Fixing one limit can simply reveal the next one.

---

## 7. `sanitized-css` Content Model Not Registered

### Symptom

XML import failed with:

```text id="u9sy95"
The content model 'sanitized-css' is not registered on this wiki.
```

### Layer

MediaWiki content model / extension layer.

### Cause

The imported dataset contained TemplateStyles CSS pages.

These use the content model:

```text id="39tb2m"
sanitized-css
```

The base MediaWiki installation did not have a handler registered for that model.

### Resolution

Install and enable TemplateStyles:

```php id="h55dc9"
wfLoadExtension( 'TemplateStyles' );
```

Then run:

```bash id="x2j1ti"
cd /var/www/mediawiki
sudo -u www-data php maintenance/run.php update
```

### Lesson

Content-model errors are strong indicators of missing MediaWiki extensions.

The import error itself can reveal which dependency is missing.

---

## 8. `{{#if: ...}}` Did Not Work

### Symptom

Imported templates containing:

```wiki id="nlilb3"
{{#if: ... }}
```

rendered incorrectly.

### Layer

MediaWiki parser extension.

### Cause

ParserFunctions had not been enabled.

### Resolution

Enable:

```php id="2mq71p"
wfLoadExtension( 'ParserFunctions' );
```

### Verification

Test:

```wiki id="u77w2q"
{{#if: test | success | failure }}
```

Expected result:

```text id="wheubx"
success
```

### Lesson

A successful page import does not imply successful template execution.

---

## 9. References Were Broken

### Symptom

Imported content using:

```wiki id="mw6p6f"
<ref>...</ref>
```

and:

```wiki id="cizq4y"
<references />
```

did not render references.

### Cause

The Cite extension was not enabled.

### Resolution

Enable:

```php id="uo1bxt"
wfLoadExtension( 'Cite' );
```

### Lesson

Unknown or broken parser tags often indicate a missing extension.

---

## 10. `hlist` Rendered Vertically

### Symptom

Navigation templates using:

```text id="qjj9h0"
hlist
```

displayed list items vertically rather than horizontally.

### Initial Assumption

The problem might be caused by a missing extension.

### Actual Cause

The required `hlist` CSS was defined in Libre Wiki's:

```text id="1m2v4a"
MediaWiki:Common.css
```

rather than by Liberty itself.

### Resolution

Copy the relevant `hlist` CSS block into the lab wiki's site-level CSS.

After the rules were added, the navigation lists rendered horizontally.

### Lesson

Not every rendering problem is an extension problem.

A useful first distinction is:

```text id="tz5i07"
syntax broken
    → parser / extension

layout broken
    → CSS / skin / gadget
```

---

## 11. Righteditlinks Was Missing

### Symptom

Libre Wiki placed section edit links on the right side of headings, while the lab did not.

### Investigation

Browser developer tools showed the ResourceLoader module:

```text id="91uddq"
ext.gadget.righteditlinks
```

### Cause

The behavior was not part of Liberty itself.

It came from a MediaWiki gadget.

### Resolution

Enable the Gadgets extension:

```php id="jhywvx"
wfLoadExtension( 'Gadgets' );
```

Register the gadget in:

```text id="a777vf"
MediaWiki:Gadgets-definition
```

and add its CSS page.

The gadget then appeared under:

```text id="g2uguy"
Preferences → Gadgets
```

and could be enabled by the user.

### Lesson

A ResourceLoader module beginning with:

```text id="9hjgoq"
ext.gadget.
```

usually points toward MediaWiki's Gadgets system.

---

## 12. Righteditlinks Appeared Below the Heading Border

### Symptom

After enabling Righteditlinks, the edit link moved to the right side but appeared below the heading separator line.

Approximate result:

```text id="fb8cue"
Heading
-------------------------
                    Edit
```

Libre Wiki instead displayed:

```text id="gjk7jh"
Heading                Edit
--------------------------
```

### Initial False Lead

Modules such as:

```text id="lemz9e"
ext.echo.styles.badge
```

appeared in developer tools.

However, Echo was unrelated.

It was merely another ResourceLoader module loaded on the page.

### Root Cause

Libre Wiki and the lab used different MediaWiki heading markup.

Libre Wiki's structure was similar to:

```html id="dr95yo"
<h2>
  <span class="mw-headline">Heading</span>
  <span class="mw-editsection">Edit</span>
</h2>
```

The MediaWiki 1.46 lab used:

```html id="bgdmcl"
<div class="mw-heading mw-heading2">
  <h2>Heading</h2>
  <span class="mw-editsection">Edit</span>
</div>
```

The old Righteditlinks CSS assumed that:

```text id="9fg2rf"
.mw-editsection
```

was inside the heading element.

It no longer was.

### Original Gadget Rule

Libre Wiki's gadget included:

```css id="dn352v"
.mw-editsection,
.mw-editsection-like {
  float: right;
  line-height: inherit;
  margin-top: 0.6em;
}
```

This worked with the older DOM.

### Resolution

Adapt the layout for the new `.mw-heading` wrapper.

Conceptually:

```css id="oxs8kw"
.mw-heading {
  display: flex;
  align-items: baseline;
}

.mw-heading > h2,
.mw-heading > h3,
.mw-heading > h4,
.mw-heading > h5,
.mw-heading > h6 {
  flex: 1;
}

.mw-heading .mw-editsection,
.mw-heading .mw-editsection-like {
  float: none;
  margin-left: auto;
  margin-top: 0;
}
```

### Lesson

Copying a customization from another MediaWiki deployment may require porting it to a newer DOM structure.

---

## 13. Heading Fix Produced Excessive Spacing

### Symptom

An early compatibility attempt moved multiple heading properties to `.mw-heading2`.

The edit link moved closer to the intended location, but large vertical gaps appeared around headings.

### Cause

Too many layout properties were moved at once.

The compatibility rule changed:

- margin
- padding
- border
- overflow behavior

instead of changing only the properties required for the new DOM.

### Resolution

Reduce the patch to the minimum required layout changes.

### Lesson

When fixing compatibility issues, avoid replacing a large styling block unless necessary.

Prefer:

> smallest possible override

over:

> recreate the entire original layout.

---

## 14. Two Heading Separator Lines Appeared

### Symptom

After moving the separator border to `.mw-heading2`, the page showed two horizontal lines.

### Cause

Both elements still had borders:

```text id="9as6d2"
h2
```

and:

```text id="59yoqr"
.mw-heading2
```

### Resolution

Keep the border on the wrapper and explicitly remove it from the inner heading:

```css id="0lpoak"
.Liberty .content-wrapper .liberty-content .liberty-content-main .mw-heading2 {
  border-bottom: 1px dashed #e1e8ed;
}

.Liberty
  .content-wrapper
  .liberty-content
  .liberty-content-main
  .mw-heading2
  > h2 {
  border-bottom: none !important;
}
```

### Why `!important` Was Needed

The Liberty skin used a more specific selector for its original heading border.

A simpler override lost the CSS specificity contest.

### Lesson

If a CSS override appears to have no effect, inspect:

- selector specificity
- inheritance
- source order
- `!important`
- computed style

before assuming the selector is wrong.

---

## 15. Browser User-Agent Styles Could Not Be Edited

### Symptom

Developer tools showed:

```css id="v3t8gc"
display: inline;
```

under:

```text id="0kmshv"
user agent stylesheet
```

The rule could not be edited directly.

### Explanation

A user-agent stylesheet is the browser's own default styling.

It is not part of MediaWiki, Liberty, or the site configuration.

### Resolution

Do not attempt to edit the user-agent stylesheet.

Instead, define an overriding site rule if the browser default actually needs to be changed.

### Lesson

Developer tools show rules from several origins:

```text id="l1dnbk"
browser defaults
MediaWiki core
skin CSS
site CSS
gadgets
inline styles
```

The source of a visible property matters.

---

## 16. ResourceLoader Module Names Were Misleading

### Symptom

While inspecting an element, modules such as:

```text id="gp4ow8"
ext.echo.styles.badge
```

appeared, which initially suggested Echo might control the layout.

### Cause

MediaWiki ResourceLoader loads many modules on the same page.

A module being present does not mean it contributes the CSS property currently being investigated.

### Better Diagnostic Method

Inspect the actual rule in the browser's:

```text id="b7hbkp"
Styles
```

or:

```text id="d4l7bl"
Computed
```

panel.

Determine which selector specifically sets:

- `display`
- `float`
- `position`
- `margin`
- `border`
- `flex`
- `right`
- `top`

### Lesson

Trace a CSS declaration, not merely a loaded resource name.

---

## 17. Common.css Could Not Be Blindly Copied

### Symptom

Copying all of Libre Wiki's `MediaWiki:Common.css` did not automatically reproduce every visual detail.

### Cause

Some rules were designed for:

- older MediaWiki markup
- different Liberty versions
- optional gadgets
- extension-generated DOM
- site-specific assumptions

### Resolution

Import relevant blocks individually and verify their effect.

Recommended process:

```text id="51d2qe"
Find missing behavior
    ↓
Find matching original CSS block
    ↓
Add only that block
    ↓
Test
    ↓
Adapt if needed
```

### Lesson

Configuration migration is not necessarily configuration copying.

---

## 18. Original Liberty Files Should Not Be Modified Directly

### Risk

A quick fix could be made by editing files under:

```text id="tv88ad"
/var/www/mediawiki/skins/Liberty
```

### Problem

Doing so would:

- complicate future `git pull`
- obscure local changes
- make upgrades harder
- mix project patches with upstream code

### Preferred Resolution

Keep local compatibility changes in:

- `MediaWiki:Common.css`
- gadget CSS
- dedicated compatibility files
- Git documentation

### Lesson

Maintain a clean boundary between upstream source and local customization.

---

## 19. CLI PHP and PHP-FPM Configuration Differ

### Symptom

A value checked with:

```bash id="u1ynj9"
php -i
```

did not necessarily describe the behavior of web requests.

### Cause

PHP CLI and PHP-FPM use separate configuration trees.

CLI:

```text id="58krfe"
/etc/php/8.5/cli/php.ini
```

FPM:

```text id="tr14b3"
/etc/php/8.5/fpm/php.ini
```

### Resolution

When troubleshooting browser uploads or MediaWiki requests, inspect the FPM configuration.

### Lesson

Always identify the execution environment before reading its configuration.

---

## 20. Configuration Changes Did Not Apply Until Service Reload

### Symptom

A corrected Nginx or PHP setting appeared unchanged in the browser.

### Cause

The running service had not reloaded the modified configuration.

### Correct Workflow

For Nginx:

```bash id="2lytfv"
sudo nginx -t
sudo systemctl reload nginx
```

For PHP-FPM:

```bash id="wgpmvw"
sudo systemctl restart php8.5-fpm
```

### Lesson

Editing a configuration file does not necessarily change a running process.

## 21. Scribunto Lua interpreter exited with status 126

### Symptom

```text
Lua error: Internal error: The interpreter exited with status 126
```

### Cause

Scribunto's bundled Lua binary was incompatible with the ARM64 environment.

### Resolution

Installed the system Lua 5.1 interpreter and configured Scribunto to use it instead of the bundled binary.

```php
$wgScribuntoEngineConf['luastandalone']['luaPath'] = '/usr/bin/lua5.1';
```

## 22. APT repository "Release file is not valid yet"

### Symptom

```text
invalid for another 7h 48min 30s. Updates for this repository will not be applied.
```

### Cause

The Ubuntu VM's system clock was out of sync with the actual time.

### Resolution

Checked the system time with `timedatectl` and re-synchronized it using NTP/chrony.

### Lesson

An APT repository error may look like a problem with the repository itself, but the actual cause can be an incorrect system clock.

---

## Troubleshooting by Layer

A useful first question is:

> Which component produced the error?

### Network Layer

Check:

```bash id="qht7si"
ip addr
ip route
ping -c 4 1.1.1.1
ping -c 4 ubuntu.com
```

### SSH Layer

Check:

```bash id="cp6ib7"
systemctl status ssh
journalctl -u ssh
```

### Nginx Layer

Check:

```bash id="u57wht"
sudo nginx -t
systemctl status nginx
sudo tail -n 50 /var/log/nginx/error.log
```

### PHP-FPM Layer

Check:

```bash id="llza31"
systemctl status php8.5-fpm
sudo journalctl -u php8.5-fpm -n 50
```

### MariaDB Layer

Check:

```bash id="itc1x9"
systemctl status mariadb
sudo journalctl -u mariadb -n 50
sudo mariadb
```

### MediaWiki Layer

Check:

```text id="17atll"
Special:Version
Special:RecentChanges
Special:Import
```

and application behavior.

### Frontend Compatibility Layer

Use browser developer tools to inspect:

- DOM
- classes
- computed CSS
- ResourceLoader modules
- parent/child relationships

---

## Error Classification Cheat Sheet

### HTTP 4xx/5xx before MediaWiki output

Likely:

```text id="fh28gh"
Nginx
network
request limits
```

### PHP error

Likely:

```text id="e1cfrr"
PHP-FPM
missing PHP module
runtime configuration
```

### Database connection error

Likely:

```text id="llosxt"
MariaDB
credentials
permissions
socket/network
```

### Unknown content model

Likely:

```text id="n7ec5e"
missing MediaWiki extension
```

### Parser syntax displayed literally

Likely:

```text id="5wtn0n"
ParserFunctions
Scribunto
other parser extension
```

### Unknown XML-style tag

Likely:

```text id="fb4lwm"
Cite
SyntaxHighlight
other tag extension
```

### Layout looks wrong but content exists

Likely:

```text id="1vbqt4"
CSS
skin
TemplateStyles
gadget
DOM version difference
```

---

## Recommended Troubleshooting Workflow

The most useful workflow discovered during this project is:

```text id="c08kph"
1. Reproduce the problem consistently
2. Identify the failing layer
3. Read the exact error message
4. Check logs or developer tools
5. Change one variable
6. Test again
7. Record the result
8. Only then move to the next hypothesis
```

Avoid changing several unrelated settings at once.

Otherwise, even if the problem disappears, it may be unclear which change solved it.

---

## Keep Working and Known-Good States

Before larger changes, record the current known-good state.

Useful commands include:

```bash id="id12yo"
git status
```

for project files, and eventually VM snapshots for infrastructure changes.

Before:

- MediaWiki upgrades
- Liberty updates
- extension upgrades
- database migration
- network restructuring

create backups or snapshots where practical.

This will become more important as the homelab grows.

---

## Useful Logs

### Nginx

```text id="o4e4xc"
/var/log/nginx/access.log
/var/log/nginx/error.log
```

### SSH

```bash id="r67lch"
journalctl -u ssh
```

### PHP-FPM

```bash id="xl4hl8"
journalctl -u php8.5-fpm
```

### MariaDB

```bash id="495q7l"
journalctl -u mariadb
```

### Live Monitoring

Examples:

```bash id="wbgu5p"
sudo tail -f /var/log/nginx/error.log
```

```bash id="6k2tyi"
sudo journalctl -u php8.5-fpm -f
```

```bash id="k4qmsh"
sudo journalctl -u mariadb -f
```

---

## Browser Developer Tools

Frontend compatibility work made browser developer tools an important part of the homelab.

Useful tasks include:

### Inspect Element

Compare the original Libre Wiki DOM with the lab DOM.

### Styles

Find the exact CSS selector responsible for a property.

### Computed

Inspect the final resolved values for:

```text id="a1tvt7"
display
position
float
margin
padding
border
line-height
```

### ResourceLoader

Inspect `load.php?...modules=...` requests to identify possible:

- skins
- extensions
- gadgets
- site styles

However, module presence alone is not proof of responsibility.

---

## Documenting Troubleshooting

Each significant issue should ideally record:

```text id="ae8pjm"
Symptom
Layer
Cause
Investigation
Resolution
Verification
Lesson
```

For example:

```text id="p4lk4n"
Symptom:
Righteditlinks appears below heading border.

Layer:
Frontend compatibility.

Cause:
Old gadget expected pre-wrapper MediaWiki heading markup.

Resolution:
Adapt layout to .mw-heading and move border to wrapper.

Verification:
Edit link appears on same row above a single separator.

Lesson:
Existing wiki customizations may require porting across MediaWiki versions.
```

This format is more useful than recording only the final command.

---

## Troubleshooting as a Project Goal

Troubleshooting is not considered an accidental side activity in this homelab.

It is one of the main goals.

The environment is intentionally suitable for:

- breaking configurations
- restoring services
- comparing implementations
- testing upgrades
- reproducing migration problems
- understanding service boundaries

Future exercises will intentionally create failures.

Examples may include:

- stopping MariaDB
- changing firewall rules incorrectly
- breaking Nginx configuration
- removing a PHP extension
- changing database credentials
- simulating disk exhaustion
- restoring from backup

The objective is to move from:

```text id="u2qp38"
I know how to install it
```

to:

```text id="n3y8wd"
I understand how it fails and how to recover it
```

---

## Major Lessons So Far

### 1. Read the exact error

`413 Request Entity Too Large` immediately identifies a different class of problem than a MediaWiki parser error.

### 2. Know the layers

Nginx, PHP, MediaWiki, and MariaDB have separate configuration and failure modes.

### 3. Do not assume an error message identifies the root cause

The missing interwiki-prefix message was caused by an oversized POST request, not by the prefix itself.

### 4. Successful migration requires more than data

Wiki content depends on:

```text id="oqfbq4"
extensions
CSS
gadgets
templates
content models
skins
```

### 5. Version differences matter

A customization that works on the source wiki can fail on MediaWiki 1.46 even if the copied CSS is correct for the original environment.

### 6. Browser developer tools are infrastructure tools too

Frontend compatibility problems can require the same structured investigation as server-side problems.

### 7. Minimal changes are easier to debug

Small, isolated changes provide better evidence than large configuration replacements.

### 8. Document the cause, not just the fix

The most reusable information is why a problem occurred.

---

## Current Known-Good State

At the end of the initial troubleshooting phase, the following are working:

- Ubuntu Server VM
- SSH key authentication
- Nginx
- PHP-FPM
- MariaDB
- MediaWiki 1.46
- Liberty
- XML import
- larger XML uploads
- TemplateStyles
- ParserFunctions
- Cite
- Gadgets
- Righteditlinks
- `hlist`
- compatibility with the newer MediaWiki heading wrapper for current tested headings

This provides the baseline for future work.

## Future Troubleshooting Areas

Likely future issues include:

- VisualEditor configuration
- additional Libre Wiki gadgets
- complex TemplateStyles dependencies
- file imports
- larger content migrations
- MariaDB migration to Rocky Linux
- SELinux
- firewalld
- inter-VM networking
- backup restoration
- monitoring
- MediaWiki upgrades
- Liberty upgrades
- extension compatibility
- ARM64-specific software availability

Each should be documented using the same layer-based approach.

## Result

The initial troubleshooting stage has established a repeatable debugging method:

```text id="94nf19"
Observe
  ↓
Locate layer
  ↓
Gather evidence
  ↓
Form hypothesis
  ↓
Make minimal change
  ↓
Verify
  ↓
Document
```

This process will be reused throughout the rest of the LibreWiki Homelab project.
