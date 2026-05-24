# FeaderHub

Official plugin registry for Feader.

FeaderHub starts with **static XPath rule packs**: data-only plugins that help Feader identify page families, improve AI XPath prompts, and provide selector candidates that Feader validates with preview diagnostics.

It also contains **view plugin templates** for Feader UI extension points. These are data-only examples for app-wide themes, source-list views, and article-detail views. Current Feader builds can use them as official template assets while the install/runtime path for remote view plugins is finalized.

Static XPath packs are the simple plugin layer. They do not execute code.

Future advanced plugins may add executable adapters, but those require a stricter permission, signing, and sandbox model.

## Repository Layout

- `registry/index.json` — official registry index.
- `plugins/*/manifest.json` — plugin metadata, permissions, integrity fields, and entry points.
- `plugins/*/xpath-rule-pack.json` — static XPath/AI rules consumed by Feader.
- `plugins/*/view-plugin.json` — data-only view plugin templates for UI extension slots.
- `schemas/*.schema.json` — JSON schema for manifests and static XPath packs.

## Trust Model

The initial registry records SHA-256 checksum fields as placeholders. Feader Core should verify checksums before installing remote packs once remote registry loading is enabled.

Planned trust layers:

- Registry checksum: verifies file integrity.
- Registry signature: verifies the official index.
- Plugin signature: verifies publisher identity.
- Permission review: users must approve expanded permissions during updates.

## Plugin Tiers

1. Static XPath rule pack
   - Selector candidates
   - AI prompt rules
   - Detection markers
   - No executable code

2. Advanced source plugin
   - Script/WASM/native-adapter execution
   - Domain permissions
   - Credential boundaries
   - Stronger sandboxing and signing

3. View plugin template
   - App UI theme tokens
   - Source-list layout metadata
   - Article-detail layout metadata
   - No executable code
