# 07. Importing Libre Wiki Content

This document describes how test pages from Libre Wiki were imported into the LibreWiki Homelab and how import failures were used to identify missing MediaWiki dependencies.

The goal of this stage was not to create a complete mirror immediately.

Instead, the strategy was:

```text
Import a small number of pages
        ↓
Observe rendering and import failures
        ↓
Identify missing extensions, CSS, gadgets, or content models
        ↓
Add the missing dependency
        ↓
Retry
```

This incremental approach made it possible to understand how Libre Wiki content depends on the surrounding MediaWiki environment.

## Environment

Current application stack:

- Ubuntu Server 26.04 LTS
- Nginx
- PHP-FPM 8.5
- MariaDB
- MediaWiki 1.46
- Liberty skin

At the beginning of this stage, the lab wiki was already functional and visually close enough to Libre Wiki for testing imported content.

## Export Source

Pages were exported from Libre Wiki using MediaWiki's built-in:

```text
Special:Export
```

interface.

For the first tests, only a small number of pages were selected.

The intent was to avoid introducing too many unknown dependencies at once.

A typical test set included:

```text
normal article
related templates
category pages
site CSS pages
```

where appropriate.

## Why Start Small?

Importing an entire wiki immediately would make troubleshooting difficult.

A single page may depend on:

- templates
- modules
- parser functions
- CSS
- gadgets
- extensions
- files
- custom namespaces

If hundreds or thousands of pages fail at the same time, it becomes difficult to determine which dependency caused which failure.

A small import makes the dependency chain easier to understand.

## XML Export

MediaWiki exports pages as XML.

The resulting file contains page and revision data rather than a database-level SQL dump.

A simplified flow is:

```text
Libre Wiki
    │
    │ Special:Export
    ▼
MediaWiki XML
    │
    ▼
LibreWiki Homelab
```

The XML file can include page revision history depending on the export options selected.

## Import Destination

The lab wiki uses:

```text
Special:Import
```

to import MediaWiki XML.

The importer is available to users with the required import permissions.

For this homelab, the administrator account was used.

## First Import Failure: HTTP 413

The first major problem occurred before MediaWiki processed the XML.

The browser displayed:

```text
413 Request Entity Too Large
```

This was an Nginx-level error.

The request had been rejected by the web server because the XML upload exceeded the configured request body size.

## Increasing the Nginx Request Limit

The LibreWiki Nginx site configuration was edited:

```bash
sudo nano /etc/nginx/sites-available/librewiki
```

A request body limit was added inside the relevant `server` block:

```nginx
client_max_body_size 50M;
```

After editing:

```bash
sudo nginx -t
```

was used to verify the configuration.

Then Nginx was reloaded:

```bash
sudo systemctl reload nginx
```

This allowed the request to pass through Nginx.

## Second Import Failure: Session Data Lost

After increasing the Nginx limit, the next import failed with messages similar to:

```text
Session data was lost.
You may have been logged out.
```

and:

```text
No interwiki prefix was supplied.
```

The interwiki prefix field had actually been filled in.

This suggested that the POST request itself was not reaching MediaWiki intact.

## Why the Interwiki Error Was Misleading

The problem was not necessarily that the entered prefix was invalid.

A more likely sequence was:

```text
Browser sends large POST request
        ↓
Nginx accepts it
        ↓
PHP rejects or truncates the POST request
        ↓
MediaWiki receives incomplete form data
        ↓
CSRF/session information is missing
        ↓
Interwiki prefix also appears missing
```

The second error was therefore a symptom rather than the root cause.

## Checking PHP Upload Limits

PHP-FPM has its own independent limits.

The relevant configuration file was:

```text
/etc/php/8.5/fpm/php.ini
```

Important values include:

```ini
upload_max_filesize
post_max_size
memory_limit
max_execution_time
max_input_time
```

These can be inspected with:

```bash
grep -E '^(upload_max_filesize|post_max_size|max_input_time|max_execution_time|memory_limit)' /etc/php/8.5/fpm/php.ini
```

For larger XML imports, values were increased to provide sufficient headroom.

For example:

```ini
upload_max_filesize = 64M
post_max_size = 80M
memory_limit = 256M
max_execution_time = 300
max_input_time = 300
```

The exact values may change later as the import workload grows.

## Restarting PHP-FPM

After modifying:

```text
/etc/php/8.5/fpm/php.ini
```

PHP-FPM was restarted:

```bash
sudo systemctl restart php8.5-fpm
```

Nginx was also reloaded where necessary:

```bash
sudo systemctl reload nginx
```

After both Nginx and PHP limits were large enough, the XML request reached MediaWiki normally.

## Upload Limits Exist at Multiple Layers

This failure illustrated that import size is controlled by multiple independent layers:

```text
Browser
   ↓
Nginx
   │ client_max_body_size
   ↓
PHP-FPM
   │ post_max_size
   │ upload_max_filesize
   ↓
MediaWiki
```

Increasing only one limit may simply reveal the next limit.

This became an important troubleshooting pattern for the project.

## Interwiki Prefix

When importing revisions from another wiki, MediaWiki asks for an interwiki prefix.

For this project, a prefix such as:

```text
librewiki
```

is appropriate.

The purpose is to distinguish users or revision origins from the source wiki.

The exact choice is less important than keeping it consistent.

## Third Import Failure: `sanitized-css`

Once request-size issues were resolved, MediaWiki began processing the actual XML.

The next error was:

```text
The content model 'sanitized-css' is not registered on this wiki.
```

This revealed a content model dependency.

## What `sanitized-css` Means

Some imported pages contained CSS used by MediaWiki TemplateStyles.

These pages are not treated as ordinary wikitext.

They use a specific content model:

```text
sanitized-css
```

The lab MediaWiki installation did not yet know how to handle that content model.

## Installing TemplateStyles

The missing dependency was the TemplateStyles extension.

It was installed under:

```text
/var/www/mediawiki/extensions/TemplateStyles
```

and enabled in:

```text
LocalSettings.php
```

with:

```php
wfLoadExtension( 'TemplateStyles' );
```

After enabling the extension, the MediaWiki maintenance update was run:

```bash
cd /var/www/mediawiki
sudo -u www-data php maintenance/run.php update
```

The extension was then confirmed through:

```text
Special:Version
```

After TemplateStyles was enabled, the `sanitized-css` content model became available and the import could proceed beyond that point.

## Imported Content May Still Render Incorrectly

A successful XML import does not guarantee that the imported page will render correctly.

For example:

```text
Page imported successfully
        ↓
Template syntax appears broken
```

or:

```text
Page imported successfully
        ↓
Layout differs from Libre Wiki
```

This often means that the page itself exists, but one of its dependencies is missing.

## Dependency Discovery Through Rendering

The following kinds of failures were used to identify missing functionality.

### Parser Functions

Templates using syntax such as:

```wiki
{{#if: ... }}
```

did not work correctly.

This indicated that ParserFunctions was not enabled.

ParserFunctions provides functions such as:

```text
#if
#ifeq
#switch
#expr
```

Once the extension was enabled, those template expressions began working.

## References

Pages containing:

```wiki
<ref>...</ref>
```

or:

```wiki
<references />
```

did not render citations correctly.

This indicated that the Cite extension was missing.

After enabling Cite, references rendered normally.

## Horizontal Lists

Some navigation templates used the:

```text
hlist
```

CSS class.

The imported lists appeared vertically rather than horizontally.

The cause was not an extension.

The required styling existed in Libre Wiki's:

```text
MediaWiki:Common.css
```

After copying the relevant `hlist` CSS block into the lab wiki's site CSS, the layout matched the expected horizontal presentation.

This demonstrated that imported content can depend on site-level CSS in addition to PHP extensions.

## Gadgets

Some Libre Wiki interface behavior appeared to come from ResourceLoader modules such as:

```text
ext.gadget.righteditlinks
```

This indicated a dependency on the Gadgets extension and wiki-defined gadget pages.

The import process therefore expanded beyond article content into interface-level dependencies.

## MediaWiki Content Is Only One Layer

By this point, the effective dependency chain looked like:

```text
Article
   ↓
Template
   ↓
Parser functions
   ↓
TemplateStyles
   ↓
Common.css
   ↓
Gadgets
   ↓
Skin behavior
```

This is one reason the project uses an incremental compatibility approach rather than treating XML import as a one-step migration.

## Files Are Not Included Automatically

MediaWiki XML exports do not automatically make all referenced uploaded files available in the destination wiki.

An imported page may contain:

```wiki
[[File:Example.jpg]]
```

while the actual file is still absent.

This results in missing file links or placeholders.

For the early stages of this homelab, file migration was intentionally deferred.

The priority was:

```text
text
templates
revision history
CSS
extensions
gadgets
```

before dealing with file storage.

## Licensing Considerations

Libre Wiki article content is not automatically stored in this Git repository.

The homelab server may contain imported content for testing, but the public Git repository is intended primarily for:

- infrastructure configuration
- compatibility fixes
- scripts
- documentation
- operational notes

XML exports and article dumps are excluded by default.

A suitable `.gitignore` rule is:

```gitignore
*.xml
```

This also avoids accidentally publishing large content archives or attribution-sensitive material.

## Useful Import Workflow

The workflow used during this stage can be summarized as:

```text
1. Select a small set of source pages
2. Export them as MediaWiki XML
3. Import into the lab
4. Record any import errors
5. Fix infrastructure limits if necessary
6. Install missing content model handlers
7. Inspect rendering
8. Identify missing extensions
9. Identify missing CSS or gadgets
10. Retry with a larger test set
```

## Troubleshooting by Layer

Import problems can be classified by where they occur.

### Nginx Layer

Example:

```text
413 Request Entity Too Large
```

Check:

```text
client_max_body_size
```

### PHP Layer

Examples:

```text
POST data missing
session data lost after large upload
```

Check:

```text
post_max_size
upload_max_filesize
memory_limit
max_execution_time
max_input_time
```

### MediaWiki Content Model Layer

Example:

```text
The content model 'sanitized-css' is not registered
```

Check which extension provides the required content model.

### Template Parsing Layer

Examples:

```text
{{#if: ...}}
```

not working.

Check:

```text
ParserFunctions
Scribunto
other parser-related extensions
```

### Rendering Layer

Examples:

```text
hlist appears vertical
navigation box layout differs
```

Check:

```text
MediaWiki:Common.css
TemplateStyles
skin CSS
gadgets
```

This layered approach makes failures easier to classify.

## Inspecting Imported Revision History

When revision history is included in the export, imported page histories can be inspected in the destination wiki.

This is useful for verifying that the lab preserves more than the current page text.

The project is interested in:

```text
page content
revision history
structure
template dependencies
```

rather than only rendering a static copy.

## Current Import Scope

At this stage, the homelab imports only selected test pages.

It is not yet a full Libre Wiki mirror.

This limitation is intentional.

A larger import should only be attempted after:

- major extensions are identified
- basic templates render correctly
- CSS dependencies are understood
- import limits are documented
- backup procedures exist

## Future Import Automation

Manual XML export and import is sufficient for initial compatibility work.

Later stages may introduce scripts that:

```text
query source page list
        ↓
detect changed pages
        ↓
export or fetch revisions
        ↓
archive locally
        ↓
import or synchronize
```

Possible tools include:

- MediaWiki API
- Python
- scheduled systemd timers
- cron
- XML processing tools

Automation should also respect the source site's operational load.

## Possible Archive Layout

A future local archive may separate:

```text
raw source data
```

from:

```text
active lab database
```

For example:

```text
archive/
├── xml/
│   ├── 2026-08-16/
│   └── ...
└── metadata/
```

while the live MediaWiki instance remains independently modifiable.

This would allow the project to preserve source revisions while also experimenting freely in the lab.

## Import vs Fork

The current lab should not be treated as a one-time static copy.

The more useful model is:

```text
Libre Wiki
   │
   │ source content
   ▼
Archive / Import
   │
   ▼
LibreWiki Lab
   │
   ├── compatibility testing
   ├── structural experiments
   ├── category reorganization
   └── extension testing
```

The lab is therefore closer to a staging environment than a replacement production wiki.

## Lessons from This Stage

Several important lessons emerged.

### Import failures can reveal architecture

The `sanitized-css` error immediately revealed the need for TemplateStyles.

Broken `#if` syntax revealed ParserFunctions.

Broken references revealed Cite.

Broken `hlist` rendering revealed a dependency on site CSS.

This made actual imported content one of the most effective ways to discover how the original wiki is configured.

### HTTP success does not mean application success

An upload can fail at Nginx, PHP, or MediaWiki.

Each layer needs to be diagnosed separately.

### Successful import does not mean successful migration

A page can exist in the database while still being visually or functionally broken.

Migration therefore requires compatibility validation after import.

### Original configuration may need adaptation

Some Libre Wiki configuration can be copied directly.

Other parts depend on older MediaWiki markup and must be ported to MediaWiki 1.46.

This became especially clear in the Liberty and Righteditlinks compatibility work.

## Current Architecture

The system at this stage includes:

```text
MacBook Air M5
│
└── UTM
    │
    └── Ubuntu Server
        │
        ├── Nginx
        ├── PHP-FPM
        ├── MariaDB
        └── MediaWiki 1.46
              │
              ├── Liberty
              ├── TemplateStyles
              ├── ParserFunctions
              ├── Cite
              ├── Gadgets
              ├── Common.css adaptations
              └── imported Libre Wiki pages
```

This is the first point where the homelab begins to behave like a real Libre Wiki compatibility environment rather than a generic MediaWiki installation.

## Future Work

Planned import-related work includes:

- import more representative pages
- install Scribunto
- import Lua modules
- identify remaining parser dependencies
- inspect complex navigation templates
- test category trees
- reproduce more gadgets
- investigate VisualEditor
- test file and image migration
- automate XML archiving
- create repeatable import procedures
- measure import performance
- test large revision histories
- test backup and restoration after import
- document attribution and licensing requirements

## Result

This stage is complete when:

- Libre Wiki pages can be exported as XML
- the lab can accept XML uploads
- Nginx upload limits are sufficient
- PHP-FPM upload limits are sufficient
- interwiki import works
- TemplateStyles handles `sanitized-css`
- ParserFunctions works
- Cite works
- `hlist` rendering can be restored
- imported content can be used to identify additional dependencies
- content XML files remain outside the public Git repository

The next stage focuses on documenting and configuring the MediaWiki extensions required to reproduce more of the Libre Wiki environment.
