# 06. Liberty Skin

This document describes how the Liberty skin was installed and adapted for the LibreWiki Homelab.

The goal of this stage was not only to change the appearance of the wiki, but also to begin reproducing part of the original Libre Wiki user interface and identify compatibility differences between the current MediaWiki version and Libre Wiki's existing customizations.

## Environment

Application stack:

- Ubuntu Server 26.04 LTS
- Nginx
- PHP-FPM 8.5
- MariaDB
- MediaWiki 1.46

Application directory:

```text
/var/www/mediawiki
```

The wiki was already fully functional before this stage.

## Why Liberty?

Libre Wiki uses the Liberty skin.

Installing the same skin provides a more useful compatibility environment than testing imported Libre Wiki content with the default MediaWiki appearance.

This also makes it easier to identify whether a visual difference comes from:

- MediaWiki core
- the Liberty skin
- site-level CSS
- gadgets
- extensions
- version differences

The goal is therefore not simply:

> Make the lab look similar to Libre Wiki.

It is also:

> Understand which parts of Libre Wiki's interface come from which layer.

## Installing Git

The Liberty skin is maintained in a Git repository, so Git was required.

If Git is not already installed:

```bash
sudo apt install git
```

## Installing Liberty

The MediaWiki skin directory is:

```text
/var/www/mediawiki/skins
```

The Liberty repository was cloned into that directory:

```bash
cd /var/www/mediawiki/skins
sudo git clone https://github.com/librewiki/liberty-skin.git Liberty
```

The resulting path is:

```text
/var/www/mediawiki/skins/Liberty
```

Ownership was adjusted:

```bash
sudo chown -R www-data:www-data /var/www/mediawiki/skins/Liberty
```

## Enabling the Skin

The main MediaWiki configuration was edited:

```bash
sudo nano /var/www/mediawiki/LocalSettings.php
```

The following lines were added:

```php
wfLoadSkin( 'Liberty' );
$wgDefaultSkin = 'Liberty';
```

After saving the file, the wiki was refreshed in the browser.

The Liberty skin loaded successfully.

## Verifying the Skin

MediaWiki's:

```text
Special:Version
```

page can be used to verify installed skins.

Liberty should appear in the skin list.

This provides a useful way to confirm that the skin has been registered rather than relying only on visual appearance.

## First Compatibility Observation

Installing Liberty did not immediately make the lab identical to Libre Wiki.

This was expected.

Libre Wiki's current interface is the result of several layers:

```text
MediaWiki core
      ↓
Liberty skin
      ↓
extensions
      ↓
site CSS
      ↓
gadgets
      ↓
wiki-specific templates
```

This became especially apparent after importing real Libre Wiki pages.

Some elements rendered correctly immediately, while others required additional site configuration or compatibility fixes.

## Site-Level CSS

One important discovery was that some visual behavior associated with Libre Wiki is not provided by Liberty itself.

Libre Wiki also uses:

```text
MediaWiki:Common.css
```

for site-level CSS.

For example, imported navigation templates using:

```text
hlist
```

did not initially render correctly.

Lists appeared vertically.

After copying the relevant `hlist` CSS rules from Libre Wiki's `MediaWiki:Common.css`, they rendered horizontally as expected.

This demonstrated that reproducing the original site requires distinguishing between:

```text
skin styles
```

and:

```text
wiki-specific site styles
```

Copying Liberty alone is not sufficient.

## Avoiding Blind CSS Copying

Although `MediaWiki:Common.css` can be imported or copied, the whole file should not automatically be assumed to be compatible.

The lab uses a newer MediaWiki environment, so some selectors may depend on older HTML structures.

A safer workflow is:

```text
Identify broken UI
      ↓
Find relevant original CSS
      ↓
Copy only the required block
      ↓
Test
      ↓
Adapt if necessary
```

This approach was used for the `hlist` rules.

## Righteditlinks Gadget

Libre Wiki places section edit links at the right side of headings.

Initially, the lab's edit links did not match Libre Wiki.

Browser developer tools revealed a ResourceLoader module named:

```text
ext.gadget.righteditlinks
```

This led to the discovery that the behavior was implemented through MediaWiki's Gadgets system rather than directly by Liberty.

Libre Wiki's gadget definition included a `Righteditlinks` gadget.

After enabling the Gadgets extension and recreating the gadget in the lab, a new:

```text
Gadgets
```

section appeared in user preferences.

Enabling the gadget changed the section edit link position.

## Original Righteditlinks CSS

The original gadget CSS was:

```css
/* This code is forked from [[wikipedia:ko:미디어위키:Gadget-Righteditlinks.css]] */

.mw-editsection,
.mw-editsection-like {
  float: right;
  line-height: inherit;
  margin-top: 0.6em;
}

#firstHeading .mw-editsection,
#firstHeading .mw-editsection-like {
  margin-top: 0.5em;
  line-height: 1em;
}

.skin-modern #firstHeading .mw-editsection,
.skin-modern #firstHeading .mw-editsection-like {
  margin-right: 0.5em;
}
```

This CSS works with the heading markup used by the original site.

However, it did not behave correctly in the lab.

## MediaWiki Heading Markup Difference

This became the first significant compatibility problem caused by version differences.

The original Libre Wiki page used heading markup similar to:

```html
<h2>
  <span class="mw-headline">Heading</span>
  <span class="mw-editsection">Edit</span>
</h2>
```

The lab's MediaWiki 1.46 environment generated a different structure:

```html
<div class="mw-heading mw-heading2">
  <h2>Heading</h2>
  <span class="mw-editsection">Edit</span>
</div>
```

The edit link is no longer inside the `<h2>`.

It is now a sibling of the heading element inside a wrapper.

This matters because the old Liberty and gadget CSS expected both the title and edit link to share the same heading element.

## Symptom

After enabling the original Righteditlinks gadget, the section edit link appeared in the wrong vertical position.

The visual result was approximately:

```text
Heading
----------------------------
                         Edit
```

while Libre Wiki displayed:

```text
Heading                     Edit
----------------------------
```

The gadget itself was loading correctly.

The issue was that its layout assumptions no longer matched the current MediaWiki DOM.

## Why the First CSS Fix Failed

The first attempt moved heading layout properties from the inner heading to the new wrapper.

This produced excessive spacing.

Another attempt resulted in two horizontal lines because both:

```text
h2
```

and:

```text
.mw-heading2
```

were drawing borders simultaneously.

This demonstrated that compatibility fixes should modify as little layout behavior as possible.

The real requirement was:

1. place the heading title and edit link on the same row
2. remove old float behavior
3. move the visual heading border to the wrapper
4. ensure the original `h2` border is disabled

## Righteditlinks Compatibility Fix

The modern heading wrapper can be treated as a flex container.

A compatibility rule similar to the following was used:

```css
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
  line-height: inherit;
}
```

This replaces the old float-based layout with one that matches the newer wrapper structure.

## Heading Border Compatibility

Liberty originally applies heading borders directly to heading elements such as:

```text
h2
```

In the new DOM structure, the edit link sits outside that element.

The border therefore needs to belong to the wrapper instead.

For second-level headings:

```css
.Liberty .content-wrapper .liberty-content .liberty-content-main .mw-heading2 {
  border-bottom: 1px dashed #e1e8ed;
}
```

The original inner border must be disabled:

```css
.Liberty
  .content-wrapper
  .liberty-content
  .liberty-content-main
  .mw-heading2
  > h2 {
  border-bottom: none !important;
}
```

The `!important` rule was needed because Liberty's existing selector had higher specificity than the initial compatibility override.

## Result

After applying both the heading layout and border compatibility rules, the lab produced the desired layout:

```text
Heading                                      Edit
-------------------------------------------------
Content
```

The section edit link remained part of the same visual heading row and the duplicate border disappeared.

## Why This Matters

This issue was more valuable than simply copying a style sheet.

The lab revealed a compatibility boundary between:

```text
existing Libre Wiki customization
```

and:

```text
newer MediaWiki markup
```

The final solution therefore became a small port rather than a direct copy.

The debugging process involved:

- inspecting the DOM in browser developer tools
- comparing original Libre Wiki HTML with lab HTML
- tracing ResourceLoader modules
- identifying the responsible gadget
- inspecting original gadget CSS
- understanding the new heading wrapper
- adapting layout behavior
- resolving CSS specificity
- eliminating duplicate borders

This became the first example in the project where an original Libre Wiki customization had to be actively adapted for a newer MediaWiki environment.

## Browser Developer Tools

Browser developer tools were particularly useful during this stage.

When examining a section edit link, several kinds of information helped isolate the problem:

- DOM hierarchy
- computed styles
- CSS selector source
- ResourceLoader module names
- inherited styles
- browser user-agent styles

One important lesson was that seeing a module such as:

```text
ext.echo.styles.badge
```

in loaded resources does not mean that module controls the selected element.

MediaWiki's ResourceLoader may bundle or load many modules on the same page.

The relevant question is:

> Which CSS declaration is actually controlling the property being investigated?

For section edit links, the responsible module was ultimately:

```text
ext.gadget.righteditlinks
```

rather than Echo.

## User Agent Styles

Developer tools also displayed browser-defined rules such as:

```css
display: inline;
```

under:

```text
user agent stylesheet
```

These rules are browser defaults and are not directly editable as site CSS.

They were not the cause of the heading issue.

Site behavior should instead be traced through the rules defined by MediaWiki, Liberty, gadgets, or site styles.

## Liberty and MediaWiki Version Compatibility

This project currently uses:

```text
MediaWiki 1.46
```

with the current Liberty skin source.

Libre Wiki itself may not use exactly the same MediaWiki version and frontend markup.

Therefore, visual compatibility should not be assumed even when the same Liberty skin is installed.

The compatibility model is closer to:

```text
Libre Wiki
├── MediaWiki version A
├── Liberty customization
├── Common.css
└── Gadgets
```

versus:

```text
LibreWiki Homelab
├── MediaWiki 1.46
├── current Liberty
├── imported/adapted Common.css
└── ported Gadgets
```

Any difference between these layers can change final rendering.

## Separating Original and Compatibility CSS

For maintainability, compatibility changes should ideally not be mixed blindly into imported site CSS.

A useful repository structure is:

```text
mediawiki/
├── common/
│   └── imported-common.css
└── compatibility/
    └── liberty-mediawiki-1.46.css
```

This allows the project to distinguish:

```text
CSS copied from Libre Wiki
```

from:

```text
CSS written specifically for the homelab
```

This distinction is useful for:

- licensing
- debugging
- future upgrades
- documentation
- understanding which behavior belongs to the original site

## Updating Liberty

Because Liberty was installed using Git, the repository can be inspected with:

```bash
cd /var/www/mediawiki/skins/Liberty
git status
```

Remote information:

```bash
git remote -v
```

Future updates should be approached carefully.

Updating the skin may alter:

- selectors
- layout
- ResourceLoader modules
- compatibility behavior

Any custom compatibility CSS should therefore be tested after a skin update.

## Do Not Modify Upstream Files Directly

Where possible, compatibility changes should not be made by editing files inside:

```text
/var/www/mediawiki/skins/Liberty
```

directly.

Direct modifications make future Git updates harder and obscure which changes came from the project.

Instead, project-specific fixes should preferably live in:

- `MediaWiki:Common.css`
- gadget CSS
- dedicated compatibility files
- documented configuration

This preserves a clearer boundary between upstream Liberty and homelab modifications.

## Current UI Layers

At the end of this stage, the lab UI can be understood as:

```text
MediaWiki 1.46
      │
      ▼
Liberty
      │
      ├── base layout
      └── base skin CSS
      │
      ▼
MediaWiki:Common.css
      │
      └── site-level rules such as hlist
      │
      ▼
Gadgets
      │
      └── Righteditlinks
      │
      ▼
Compatibility CSS
      │
      └── new heading DOM adaptation
```

This layered model became important for later troubleshooting.

## Planned Liberty Work

Further work may include:

- compare more Libre Wiki site CSS
- reproduce additional gadgets
- inspect mobile behavior
- compare navigation layout
- investigate sidebar configuration
- test higher-level headings
- test tables and navigation boxes
- verify dark-mode behavior if applicable
- document Liberty upgrades
- contribute compatibility fixes upstream if appropriate

## Current Architecture

The system now looks like:

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
              ├── Common.css
              ├── Gadgets
              └── compatibility overrides
```

## Result

This stage is complete when:

- Liberty is installed
- Liberty is enabled as the default skin
- the skin appears under `Special:Version`
- site-level CSS can be distinguished from skin CSS
- `hlist` styling is restored
- the Gadgets system is available
- Righteditlinks can be enabled
- the original Righteditlinks behavior is understood
- the MediaWiki heading DOM difference is identified
- the section edit link is adapted to the newer markup
- duplicate heading borders are removed
- project-specific compatibility fixes are kept separate from upstream Liberty where possible

The next stage focuses on importing real Libre Wiki content and using import failures as a way to discover additional extensions and dependencies.
