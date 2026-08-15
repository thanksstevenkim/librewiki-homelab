# 08. MediaWiki Extensions

This document describes the MediaWiki extensions required so far to reproduce Libre Wiki content and interface behavior in the LibreWiki Homelab.

The goal of this stage was not to install a large list of extensions in advance.

Instead, extensions were added only when real imported content or interface behavior showed that they were required.

The working method was:

```text
Import or reproduce real Libre Wiki behavior
        ↓
Observe what fails
        ↓
Identify which extension provides that feature
        ↓
Install and enable it
        ↓
Verify the result
```

This made extension setup part of the compatibility investigation rather than a separate checklist.

## Environment

Current application stack:

- Ubuntu Server 26.04 LTS
- Nginx
- PHP-FPM 8.5
- MariaDB
- MediaWiki 1.46
- Liberty skin

MediaWiki application directory:

```text
/var/www/mediawiki
```

Extension directory:

```text
/var/www/mediawiki/extensions
```

Main configuration file:

```text
/var/www/mediawiki/LocalSettings.php
```

## Extension Strategy

The project intentionally avoids installing every extension that might exist on Libre Wiki.

There are several reasons for this.

Installing unnecessary extensions can:

- increase complexity
- introduce compatibility problems
- make troubleshooting harder
- hide which imported feature actually depends on which component
- increase upgrade and security maintenance work

The preferred approach is:

```text
minimal MediaWiki
      ↓
real imported content
      ↓
missing feature
      ↓
specific extension
```

This keeps the environment understandable.

## Checking Installed Extensions

MediaWiki provides:

```text
Special:Version
```

which displays installed and enabled extensions.

This page is useful for confirming that an extension was actually loaded.

A directory existing under:

```text
extensions/
```

does not necessarily mean the extension is active.

Activation normally requires a line such as:

```php
wfLoadExtension( 'ExtensionName' );
```

in `LocalSettings.php`.

## Database Updates

Some extensions require database schema changes.

After installing or enabling such an extension, the MediaWiki maintenance updater can be run with:

```bash
cd /var/www/mediawiki
sudo -u www-data php maintenance/run.php update
```

This became a standard post-installation step where appropriate.

## TemplateStyles

### Why It Was Needed

During XML import, MediaWiki produced:

```text
The content model 'sanitized-css' is not registered on this wiki.
```

This indicated that imported pages were using a content model that the base installation did not recognize.

The missing handler was provided by TemplateStyles.

### What TemplateStyles Provides

TemplateStyles allows templates and related pages to use scoped, sanitized CSS.

Pages using this system can have the content model:

```text
sanitized-css
```

Templates may reference these styles through syntax such as:

```wiki
<templatestyles src="Template:Example/styles.css" />
```

### Installation

TemplateStyles was installed under:

```text
/var/www/mediawiki/extensions/TemplateStyles
```

and enabled with:

```php
wfLoadExtension( 'TemplateStyles' );
```

The maintenance updater was then executed:

```bash
cd /var/www/mediawiki
sudo -u www-data php maintenance/run.php update
```

### Verification

The extension was verified through:

```text
Special:Version
```

After activation, XML pages using the `sanitized-css` content model could be imported.

### Lesson

This was the first extension discovered directly through an import failure.

The error itself identified the missing architectural dependency.

---

## ParserFunctions

### Why It Was Needed

Imported templates contained parser function syntax such as:

```wiki
{{#if: ... }}
```

Without ParserFunctions, this syntax did not behave correctly.

### What ParserFunctions Provides

ParserFunctions provides logic and expression functions commonly used by complex MediaWiki templates.

Examples include:

```wiki
{{#if: ... }}
{{#ifeq: ... }}
{{#switch: ... }}
{{#expr: ... }}
```

These allow templates to contain conditional logic and calculations.

### Enabling ParserFunctions

If the extension directory already exists:

```bash
ls /var/www/mediawiki/extensions/ParserFunctions
```

it can be enabled in `LocalSettings.php`:

```php
wfLoadExtension( 'ParserFunctions' );
```

The maintenance updater can then be run:

```bash
cd /var/www/mediawiki
sudo -u www-data php maintenance/run.php update
```

### Verification

A simple test page can contain:

```wiki
{{#if: test | success | failure }}
```

Expected result:

```text
success
```

### Why It Matters

ParserFunctions is heavily used by reusable templates.

A wiki can successfully import template pages while still rendering them incorrectly if parser functions are unavailable.

---

## Cite

### Why It Was Needed

Imported articles used references such as:

```wiki
<ref>Reference text</ref>
```

and:

```wiki
<references />
```

These did not render correctly before Cite was enabled.

### What Cite Provides

Cite provides MediaWiki's footnote and reference system.

It handles:

- `<ref>`
- `<references>`
- named references
- repeated citations
- reference groups

### Enabling Cite

If the extension is included in the MediaWiki installation:

```bash
ls /var/www/mediawiki/extensions/Cite
```

it can be enabled with:

```php
wfLoadExtension( 'Cite' );
```

Then:

```bash
cd /var/www/mediawiki
sudo -u www-data php maintenance/run.php update
```

can be run if required.

### Verification

A test page can contain:

```wiki
This is a test sentence.<ref>This is a test reference.</ref>

== References ==
<references />
```

A numbered reference should appear in the text and the reference content should appear below.

---

## Gadgets

### Why It Was Needed

Libre Wiki interface behavior included ResourceLoader modules such as:

```text
ext.gadget.righteditlinks
```

This indicated that some functionality was implemented using MediaWiki's Gadgets system rather than the Liberty skin itself.

### What Gadgets Provides

Gadgets allows site administrators to define optional CSS and JavaScript features that users can enable through preferences.

Typical components include:

```text
MediaWiki:Gadgets-definition
MediaWiki:Gadget-Example.css
MediaWiki:Gadget-Example.js
```

### Enabling Gadgets

If the extension is available:

```bash
ls /var/www/mediawiki/extensions/Gadgets
```

it can be enabled in `LocalSettings.php`:

```php
wfLoadExtension( 'Gadgets' );
```

After activation, a:

```text
Gadgets
```

section appears in user preferences.

### Gadget Definitions

Gadgets are registered through:

```text
MediaWiki:Gadgets-definition
```

For example, the Righteditlinks gadget can be defined with a line similar to:

```text
* Righteditlinks[ResourceLoader]|Righteditlinks.css
```

The corresponding page is:

```text
MediaWiki:Gadget-Righteditlinks.css
```

### User Activation

Unlike `MediaWiki:Common.css`, gadgets may be optional per-user features.

After registering Righteditlinks, the feature did not become active automatically.

It became available under:

```text
Preferences → Gadgets
```

and had to be enabled there.

### Lesson

Seeing a ResourceLoader module beginning with:

```text
ext.gadget.
```

is a useful indication that the behavior may come from a MediaWiki gadget.

---

## Righteditlinks

Righteditlinks is not a standalone PHP extension.

It is a gadget running through the Gadgets extension.

Libre Wiki's version contained CSS similar to:

```css
.mw-editsection,
.mw-editsection-like {
  float: right;
  line-height: inherit;
  margin-top: 0.6em;
}
```

This moved section edit links to the right side of headings.

However, the original CSS expected older MediaWiki heading markup.

The MediaWiki 1.46 lab environment uses a newer heading wrapper structure, so the gadget required compatibility changes.

Those changes are documented in:

```text
docs/06-liberty.md
```

This distinction is important:

```text
Gadgets
   └── extension framework

Righteditlinks
   └── wiki-defined gadget
```

---

## Site CSS Is Not an Extension

Some missing behavior initially looked like it might require another extension.

For example, imported navigation boxes using:

```text
hlist
```

rendered vertically.

The actual cause was missing site CSS from:

```text
MediaWiki:Common.css
```

rather than a missing extension.

The relevant rule had to be copied or adapted from Libre Wiki.

This created an important troubleshooting distinction:

```text
broken parser syntax
        ↓
probably extension

broken layout
        ↓
possibly CSS / skin / gadget
```

This is not absolute, but it is a useful first diagnostic step.

---

## Extension vs Gadget vs Site CSS

By this point, Libre Wiki compatibility behavior could be classified into several layers.

### Extension

Loaded through PHP configuration:

```php
wfLoadExtension( 'ParserFunctions' );
```

Examples:

- TemplateStyles
- ParserFunctions
- Cite
- Gadgets

### Gadget

Defined inside wiki pages and executed through the Gadgets extension.

Example:

```text
Righteditlinks
```

### Site CSS

Defined through pages such as:

```text
MediaWiki:Common.css
```

Example:

```text
hlist
```

### Skin

Installed under:

```text
skins/
```

Example:

```text
Liberty
```

Understanding these layers prevents unnecessary extension installation.

---

## Current Required Extensions

At the current stage, the LibreWiki Homelab requires at least:

```text
TemplateStyles
ParserFunctions
Cite
Gadgets
```

The exact set will grow as more complex Libre Wiki pages are imported.

Current dependency view:

```text
MediaWiki
│
├── TemplateStyles
│   └── sanitized-css
│
├── ParserFunctions
│   └── #if / #switch / expressions
│
├── Cite
│   └── references
│
└── Gadgets
    └── Righteditlinks
```

---

## Likely Next Extension: Scribunto

One likely future dependency is Scribunto.

Libre Wiki templates may contain:

```wiki
{{#invoke:ModuleName|function}}
```

This syntax requires Scribunto.

Scribunto allows Lua code stored in:

```text
Module:
```

namespace pages to be executed by templates.

The dependency chain would then look like:

```text
Article
   ↓
Template
   ↓
#invoke
   ↓
Scribunto
   ↓
Lua module
```

Scribunto has not yet been configured at the point documented here.

It should be added only when imported content demonstrates the need.

---

## VisualEditor

Libre Wiki appears to offer separate editing interfaces such as:

```text
Edit
Edit source
```

This behavior suggests VisualEditor is used.

VisualEditor is not required for basic content rendering or XML import.

It is therefore lower priority than extensions required by imported pages.

The planned order is:

```text
content compatibility first
        ↓
editing interface later
```

VisualEditor will be configured in a future stage after core rendering dependencies are stable.

---

## Dependency Discovery Workflow

When a new imported page fails, the following process is useful.

### 1. Identify the failure

Examples:

```text
{{#invoke:...}} visible
```

or:

```text
Unknown tag
```

or:

```text
content model not registered
```

### 2. Determine the layer

Possible layers include:

```text
MediaWiki core
extension
skin
site CSS
gadget
template
Lua module
```

### 3. Search `Special:Version`

Check whether the expected extension is already installed.

### 4. Inspect the source page

Look for syntax such as:

```wiki
{{#if:}}
{{#invoke:}}
<ref>
<templatestyles>
```

### 5. Install only the required dependency

Avoid enabling unrelated extensions.

### 6. Run maintenance updates if needed

```bash
sudo -u www-data php maintenance/run.php update
```

### 7. Test again

Use the same imported page to verify whether the dependency was correctly identified.

---

## Extension Installation Pattern

A typical extension installation follows this pattern.

### Install or clone

```bash
cd /var/www/mediawiki/extensions
sudo git clone <extension repository>
```

### Set ownership where appropriate

```bash
sudo chown -R www-data:www-data /var/www/mediawiki/extensions/ExtensionName
```

### Enable

In `LocalSettings.php`:

```php
wfLoadExtension( 'ExtensionName' );
```

### Update

```bash
cd /var/www/mediawiki
sudo -u www-data php maintenance/run.php update
```

### Verify

Open:

```text
Special:Version
```

and confirm that the extension appears.

Not every extension requires every one of these steps, so the extension's own documentation should still be checked.

---

## Extension Version Compatibility

The homelab uses MediaWiki 1.46.

Extensions must therefore be compatible with the installed MediaWiki version.

Blindly copying extension directories from another wiki can create problems if that wiki runs an older MediaWiki version.

Possible compatibility issues include:

- removed hooks
- changed APIs
- changed HTML structure
- schema differences
- ResourceLoader changes
- PHP version requirements

The Righteditlinks problem demonstrated that even non-PHP customizations can depend on MediaWiki version-specific frontend markup.

Extension compatibility therefore needs to be evaluated separately from content compatibility.

---

## Upstream vs Wiki-Specific Components

The project distinguishes between components maintained upstream and components defined by Libre Wiki itself.

### Upstream components

Examples:

```text
TemplateStyles
ParserFunctions
Cite
Gadgets
Liberty
```

These should normally be installed from their maintained source repositories or official MediaWiki release packages.

### Wiki-specific components

Examples:

```text
MediaWiki:Common.css
MediaWiki:Gadgets-definition
MediaWiki:Gadget-Righteditlinks.css
```

These come from the original wiki configuration and may require adaptation.

This separation makes future upgrades easier.

---

## Git Repository Organization

A useful structure for extension-related project files is:

```text
mediawiki/
├── extensions/
│   └── README.md
├── gadgets/
│   └── Righteditlinks.css
├── common/
│   └── required-common.css
└── compatibility/
    └── liberty-mediawiki-1.46.css
```

The actual upstream extension source directories do not need to be copied into this repository if they can be installed directly from their official repositories.

Instead, this repository should document:

- which extensions are required
- why they are required
- how they are enabled
- any local compatibility changes

---

## Suggested Extension Inventory

A future inventory may look like:

| Component       | Type      | Status  | Reason                        |
| --------------- | --------- | ------- | ----------------------------- |
| Liberty         | Skin      | Enabled | Libre Wiki UI                 |
| TemplateStyles  | Extension | Enabled | `sanitized-css`, template CSS |
| ParserFunctions | Extension | Enabled | `#if`, `#switch`, expressions |
| Cite            | Extension | Enabled | References                    |
| Gadgets         | Extension | Enabled | Site gadgets                  |
| Righteditlinks  | Gadget    | Enabled | Section edit link layout      |
| Scribunto       | Extension | Planned | Lua / `#invoke`               |
| VisualEditor    | Extension | Planned | Visual editing                |

This table should be expanded as new dependencies are discovered.

---

## Security Considerations

Every extension increases the amount of code running inside MediaWiki.

Extensions should therefore be:

- installed only when needed
- obtained from trusted upstream sources
- kept compatible with the MediaWiki release
- reviewed before major upgrades
- removed if no longer necessary

Unmaintained extensions should receive additional scrutiny.

The lab is not publicly exposed yet, but the same discipline should be used as if it could eventually become externally reachable.

---

## Upgrade Considerations

Future MediaWiki upgrades may break:

- extensions
- gadgets
- CSS selectors
- skin integrations

Before upgrading MediaWiki, the project should record the current compatibility state.

A future upgrade workflow might be:

```text
snapshot VM
     ↓
backup database
     ↓
upgrade MediaWiki
     ↓
upgrade extensions
     ↓
run maintenance update
     ↓
test imported pages
     ↓
test Liberty
     ↓
test gadgets
```

This is another reason for keeping compatibility modifications documented rather than making unexplained manual changes.

---

## Current Extension Architecture

At the end of this stage:

```text
MediaWiki 1.46
│
├── Liberty
│
├── TemplateStyles
│
├── ParserFunctions
│
├── Cite
│
└── Gadgets
      │
      └── Righteditlinks
```

Additional behavior comes from:

```text
MediaWiki:Common.css
```

and locally written compatibility CSS.

---

## Future Work

Planned extension work includes:

- install Scribunto
- test Lua module imports
- configure VisualEditor
- identify additional Libre Wiki extensions
- reproduce more gadgets
- investigate category-related extensions
- test syntax highlighting
- inspect complex template dependencies
- build an extension inventory
- document version compatibility
- automate extension installation where practical
- evaluate extension upgrades safely

## Result

This stage is complete when:

- TemplateStyles is enabled
- `sanitized-css` content imports successfully
- ParserFunctions works
- Cite renders references
- Gadgets is enabled
- Righteditlinks is available in preferences
- extension responsibilities are distinguished from gadgets and site CSS
- missing features can be investigated through a repeatable dependency-discovery process
- planned extensions are documented without installing them prematurely

The next stage should focus on the remaining compatibility and troubleshooting work, especially the differences between Libre Wiki's existing frontend customizations and MediaWiki 1.46.
