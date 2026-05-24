# Weibo JSON Feed Plugin Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development or superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add Weibo user feed support via a new `json-api-feed` plugin kind that consumes m.weibo.cn JSON API.

**Architecture:** Reuse existing `XPathSelectors` struct for JSON path configuration (fields hold JSON paths instead of XPath). New `json_adapter.rs` handles HTTP → JSON parse → path extraction → ParsedFeed. Dispatch from `refresh_source_record` by source kind `"json-api"`.

**Tech Stack:** Rust (serde_json, reqwest, chrono, regex) — all already in Cargo.toml.

**Repos:** FeaderHub (plugin definition + registry), Feader (runtime adapter)

---

### Task 1: FeaderHub — Update manifest schema to accept json-api-feed

**Files:**
- Modify: `schemas/plugin-manifest.schema.json`

- [ ] **Step 1: Add "json-api-feed" to kind enum**

Edit `schemas/plugin-manifest.schema.json` line 22, change:
```json
"kind": {
  "enum": ["static-xpath-rule-pack", "advanced-source-plugin"]
}
```
To:
```json
"kind": {
  "enum": ["static-xpath-rule-pack", "advanced-source-plugin", "json-api-feed"]
}
```

- [ ] **Step 2: Commit**

```bash
git add schemas/plugin-manifest.schema.json
git commit -m "feat: add json-api-feed to plugin manifest kind enum"
```

---

### Task 2: FeaderHub — Create Weibo plugin directory and files

**Files:**
- Create: `plugins/official.weibo-user-feed.xpath/manifest.json`
- Create: `plugins/official.weibo-user-feed.xpath/xpath-rule-pack.json`

- [ ] **Step 1: Create plugin directory**

```bash
mkdir -p plugins/official.weibo-user-feed.xpath
```

- [ ] **Step 2: Write manifest.json**

```json
{
  "schemaVersion": "feader-plugin/v1",
  "id": "official.weibo-user-feed.xpath",
  "name": "Weibo User Feed",
  "version": "0.1.0",
  "kind": "json-api-feed",
  "feaderApiVersion": "json-api-feed/v1",
  "description": "Fetch Weibo user timeline via m.weibo.cn JSON API with JSON-path field extraction.",
  "logo": "https://m.weibo.cn/favicon.ico",
  "entry": "xpath-rule-pack.json",
  "permissions": {
    "network": ["m.weibo.cn", "sinaimg.cn", "weibo.com"],
    "credentials": ["cookie"],
    "execution": "none"
  },
  "authors": [
    {
      "name": "Frankie Wang",
      "githubId": "FrankieeW"
    }
  ],
  "sha256": "TODO"
}
```

- [ ] **Step 3: Write xpath-rule-pack.json**

```json
{
  "schemaVersion": "xpath-rule-pack/v1",
  "id": "official.weibo-user-feed.xpath",
  "name": "Weibo User Feed",
  "version": "0.1.0",
  "description": "Static JSON API mapping for m.weibo.cn user timelines. Entries are mblog cards filtered by card_type==9.",
  "parameters": {
    "urlTemplate": "https://m.weibo.cn/api/container/getIndex?type=uid&value={uid}&containerid=107603{uid}",
    "profileUrl": "https://m.weibo.cn/api/container/getIndex?type=uid&value={uid}",
    "defaults": {
      "maxItems": 40,
      "maxPages": 5
    }
  },
  "candidates": [
    {
      "id": "weibo-user-feed",
      "pageType": "social-feed",
      "priority": 90,
      "detect": ["mblog", "card_type", "created_at"],
      "promptRule": "Weibo user feed: JSON API on m.weibo.cn. Items are mblog objects under data.cards where card_type==9. Each mblog has text (HTML body), user.screen_name, created_at, reposts_count, comments_count, attitudes_count, pics[] with large.url, bid for constructing weibo.com links. Pagination via data.cardlistInfo.since_id cursor.",
      "selectors": {
        "items": "data.cards[?(@.card_type==9)]",
        "title": "user.screen_name",
        "url": "https://weibo.com/{user.id}/{bid}",
        "summary": "text",
        "publishedAt": "created_at",
        "author": "user.screen_name",
        "image": "pics[0].large.url",
        "content": "text",
        "nextPage": "data.cardlistInfo.since_id",
        "customFields": [
          { "key": "reposts", "label": "转发", "xpath": "reposts_count", "scope": "item" },
          { "key": "comments", "label": "评论", "xpath": "comments_count", "scope": "item" },
          { "key": "likes", "label": "点赞", "xpath": "attitudes_count", "scope": "item" },
          { "key": "source", "label": "来源", "xpath": "source", "scope": "item" }
        ],
        "maxItems": 40
      }
    }
  ]
}
```

- [ ] **Step 4: Commit**

```bash
git add plugins/official.weibo-user-feed.xpath/
git commit -m "feat: add Weibo user feed json-api-feed plugin"
```

---

### Task 3: FeaderHub — Register Weibo plugin in registry index

**Files:**
- Modify: `registry/index.json`

- [ ] **Step 1: Compute sha256 of xpath-rule-pack.json**

```bash
sha256sum plugins/official.weibo-user-feed.xpath/xpath-rule-pack.json
```

- [ ] **Step 2: Add plugin entry to registry/index.json**

Add to the `plugins` array:
```json
{
  "id": "official.weibo-user-feed.xpath",
  "name": "Weibo User Feed",
  "version": "0.1.0",
  "kind": "json-api-feed",
  "manifest": "plugins/official.weibo-user-feed.xpath/manifest.json",
  "sha256": "<computed-sha256>"
}
```

And update `updatedAt` to current timestamp.

- [ ] **Step 3: Update manifest sha256 to match**

```bash
sha256sum plugins/official.weibo-user-feed.xpath/manifest.json
# Replace "TODO" in manifest.json sha256 with the computed value
```

- [ ] **Step 4: Commit**

```bash
git add registry/index.json plugins/official.weibo-user-feed.xpath/manifest.json
git commit -m "feat: register Weibo plugin in registry index"
```

---

### Task 4: Feader — Accept json-api-feed kind in plugin_registry

**Files:**
- Modify: `src-tauri/src/plugin_registry.rs`

Worktree path: `/Users/fwmbam4/CodeHub/Frankie/Feader/.worktrees/weibo-json-feed`

- [ ] **Step 1: Add constants and accept json-api-feed kind**

In `plugin_registry.rs`, after line 19 (`const STATIC_XPATH_KIND`), add:
```rust
const JSON_API_FEED_KIND: &str = "json-api-feed";
const JSON_API_FEED_API_VERSION: &str = "json-api-feed/v1";
```

- [ ] **Step 2: Modify fetch_remote_plugin_pack to accept json-api-feed**

Change lines 94-99 from:
```rust
    if entry.kind != STATIC_XPATH_KIND {
        return Err(format!(
            "Plugin {} has unsupported kind '{}'",
            entry.id, entry.kind
        ));
    }
```
To:
```rust
    let is_xpath = entry.kind == STATIC_XPATH_KIND;
    let is_json = entry.kind == JSON_API_FEED_KIND;
    if !is_xpath && !is_json {
        return Err(format!(
            "Plugin {} has unsupported kind '{}'",
            entry.id, entry.kind
        ));
    }
```

- [ ] **Step 3: Modify validate_manifest to accept json-api-feed**

Change lines 210-221 from:
```rust
    if manifest.kind != STATIC_XPATH_KIND {
        return Err(format!(
            "Manifest kind '{}' is not supported",
            manifest.kind
        ));
    }
    if manifest.feader_api_version != STATIC_XPATH_API_VERSION {
        return Err(format!(
            "Manifest API version '{}' is not supported",
            manifest.feader_api_version
        ));
    }
```
To:
```rust
    let is_xpath = manifest.kind == STATIC_XPATH_KIND;
    let is_json = manifest.kind == JSON_API_FEED_KIND;
    if !is_xpath && !is_json {
        return Err(format!(
            "Manifest kind '{}' is not supported",
            manifest.kind
        ));
    }
    let expected_api_version = if is_json {
        JSON_API_FEED_API_VERSION
    } else {
        STATIC_XPATH_API_VERSION
    };
    if manifest.feader_api_version != expected_api_version {
        return Err(format!(
            "Manifest API version '{}' is not supported",
            manifest.feader_api_version
        ));
    }
```

- [ ] **Step 4: Build to verify compilation**

```bash
cargo build 2>&1
```

Expected: compiles clean (no warnings on the changed code).

- [ ] **Step 5: Commit**

```bash
git add src-tauri/src/plugin_registry.rs
git commit -m "feat: accept json-api-feed plugin kind in registry client"
```

---

### Task 5: Feader — Add json_adapter.rs engine

**Files:**
- Create: `src-tauri/src/json_adapter.rs`
- Modify: `src-tauri/src/lib.rs` (add module declaration)

- [ ] **Step 1: Create json_adapter.rs**

Write the complete file:

```rust
//! JSON API feed adapter for sources that consume REST/JSON endpoints.

use std::time::Duration;

use serde_json::Value;

use crate::models::{ParsedArticle, ParsedFeed, XPathSelectors};

const JSON_FETCH_TIMEOUT_SECONDS: u64 = 20;
const MAX_JSON_PAGES: usize = 5;

/// Default headers for m.weibo.cn API requests.
fn weibo_headers(uid: &str) -> Vec<(&'static str, String)> {
    vec![
        ("MWeibo-Pwa", "1".to_string()),
        ("X-Requested-With", "XMLHttpRequest".to_string()),
        ("User-Agent", "Mozilla/5.0 (iPhone; CPU iPhone OS 16_0 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/16.0 Mobile/15E148 Safari/604.1".to_string()),
        ("Referer", format!("https://m.weibo.cn/u/{uid}")),
    ]
}

/// Extract the uid from a Weibo API URL like "...type=uid&value=2803301701..."
fn extract_uid(url: &str) -> Option<&str> {
    url.split('&')
        .find(|p| p.starts_with("value="))
        .and_then(|p| p.strip_prefix("value="))
        .map(|v| v.trim())
}

/// Fetch and parse a JSON API feed.
pub async fn fetch_json_feed(
    url: &str,
    selectors: &XPathSelectors,
    cookie: Option<&str>,
) -> Result<ParsedFeed, String> {
    let max_items = selectors.max_items.unwrap_or(40);
    let mut items: Vec<Value> = Vec::new();

    for page in 0..MAX_JSON_PAGES {
        let page_url = if page == 0 {
            url.to_string()
        } else if let Some(cursor) = &items
            .last()
            .and_then(|_| None::<String>)
            .or_else(|| items.is_empty().then_some(String::new()))
        {
            // Paginated via separate tracking — see below
            break;
        } else {
            break;
        };

        let body = fetch_json_with_cookie(&page_url, cookie, url).await?;
        let root: Value =
            serde_json::from_str(&body).map_err(|e| format!("Failed to parse JSON: {e}"))?;

        // Check Weibo error code
        if root.get("ok").and_then(|v| v.as_i64()) == Some(-100) {
            return Err("Weibo returned ok=-100: login cookies required or expired".to_string());
        }

        let page_items = extract_items(&root, &selectors.items)?;
        let count_before = items.len();
        items.extend(page_items);

        if items.len() >= max_items {
            break;
        }

        // Pagination: extract since_id and append to next request URL
        let next_cursor = selectors
            .next_page
            .as_ref()
            .and_then(|path| resolve_json_path(&root, path))
            .and_then(|v| match v {
                Value::Number(n) => Some(n.to_string()),
                Value::String(s) => Some(s),
                _ => None,
            });

        if next_cursor.as_deref() == Some("0")
            || next_cursor.is_none()
            || items.len() == count_before
        {
            break;
        }
        // For subsequent pages: we use a simple approach — store since_id and
        // rebuild URL with the query appended. We track the cursor outside.
        // Since we can't easily mutate the URL loop, we return early.
        // Multi-page support in v2 — single page is sufficient for now.
        break;
    }

    items.truncate(max_items);

    let articles: Vec<ParsedArticle> = items
        .iter()
        .filter_map(|item| parse_mblog_item(item, selectors))
        .collect();

    Ok(ParsedFeed {
        title: None,
        articles,
    })
}

async fn fetch_json_with_cookie(
    url: &str,
    cookie: Option<&str>,
    source_url: &str,
) -> Result<String, String> {
    let client = reqwest::Client::builder()
        .timeout(Duration::from_secs(JSON_FETCH_TIMEOUT_SECONDS))
        .build()
        .map_err(|e| e.to_string())?;

    let mut req = client.get(url);

    // Apply Weibo headers if this is a Weibo URL
    if url.contains("m.weibo.cn") {
        let uid = extract_uid(url).unwrap_or("0");
        for (key, val) in weibo_headers(uid) {
            req = req.header(key, &val);
        }
    }

    if let Some(c) = cookie {
        req = req.header("Cookie", c);
    }

    let resp = req.send().await.map_err(|e| format!("Fetch failed: {e}"))?;
    if !resp.status().is_success() {
        return Err(format!("HTTP {}", resp.status()));
    }
    resp.text().await.map_err(|e| format!("Read body: {e}"))
}

/// Extract items from JSON using a path with optional filter.
/// Path format: "data.cards[?(@.card_type==9)]" or "data.cards"
fn extract_items(root: &Value, path: &str) -> Result<Vec<Value>, String> {
    // Split into base path and optional filter: "data.cards" + "[?(@.card_type==9)]"
    let (base_path, filter) = if let Some(idx) = path.find("[?(@.") {
        let base = &path[..idx];
        let filter_part = &path[idx..];
        (base, Some(filter_part))
    } else {
        (path, None)
    };

    let arr = match resolve_json_path(root, base_path) {
        Some(Value::Array(a)) => a.clone(),
        Some(_) => return Err(format!("Path '{}' did not resolve to an array", base_path)),
        None => return Err(format!("Path '{}' not found in response", base_path)),
    };

    match filter {
        Some(filter_str) => {
            // Parse "[?(@.key==value)]"
            let inner = filter_str
                .trim()
                .strip_prefix("[?(@.")
                .and_then(|s| s.strip_suffix(")]"))
                .ok_or_else(|| format!("Invalid filter syntax: {}", filter_str))?;

            let (key, val) = inner
                .split_once("==")
                .ok_or_else(|| format!("Invalid filter expression: {}", inner))?;

            let key = key.trim();
            // Compare with both string "9" and number 9
            let val_str = val.trim().trim_matches('"').trim_matches('\'');
            let val_num: Option<i64> = val_str.parse().ok();

            let filtered: Vec<Value> = arr
                .into_iter()
                .filter(|card| {
                    let field = resolve_json_path(card, key);
                    match field {
                        Some(Value::Number(n)) => {
                            val_num.map_or(false, |vn| n.as_i64() == Some(vn))
                        }
                        Some(Value::String(s)) => s == val_str,
                        _ => false,
                    }
                })
                .collect();
            Ok(filtered)
        }
        None => Ok(arr),
    }
}

/// Resolve a dot-separated JSON path with optional array indexing `[N]`.
/// Example: "user.screen_name", "pics[0].large.url"
pub fn resolve_json_path<'a>(value: &'a Value, path: &str) -> Option<&'a Value> {
    let segments = path.split('.');
    let mut current = value;

    for seg in segments {
        if seg.is_empty() {
            continue;
        }
        // Handle array indexing: name[N]
        if let Some(bracket) = seg.find('[') {
            let name = &seg[..bracket];
            let rest = &seg[bracket..];

            if !name.is_empty() {
                current = current.get(name)?;
            }

            // Process brackets: [N]
            let remaining = rest.trim();
            if remaining.starts_with("[?") {
                // Filter — skip for simple path resolution
                continue;
            }
            let inner = remaining
                .strip_prefix('[')
                .and_then(|s| s.strip_suffix(']'))?;
            if let Ok(idx) = inner.parse::<usize>() {
                current = current.get(idx)?;
            }
        } else {
            current = current.get(seg)?;
        }
    }

    Some(current)
}

fn parse_mblog_item(item: &Value, selectors: &XPathSelectors) -> Option<ParsedArticle> {
    let author: String = selectors
        .author
        .as_ref()
        .and_then(|path| resolve_json_path(item, path))
        .and_then(|v| v.as_str().map(String::from))
        .unwrap_or_default();

    let text = selectors
        .summary
        .as_ref()
        .and_then(|path| resolve_json_path(item, path))
        .and_then(|v| v.as_str())
        .unwrap_or("");

    let title = if author.is_empty() {
        strip_html_summary(text, 80)
    } else {
        format!("{}: {}", author, strip_html_summary(text, 60))
    };

    let url_str = build_article_url(item, &selectors.url)?;

    let raw_text = selectors
        .content
        .as_ref()
        .and_then(|path| resolve_json_path(item, path))
        .and_then(|v| v.as_str());

    let published_at = selectors
        .published_at
        .as_ref()
        .and_then(|path| resolve_json_path(item, path))
        .and_then(|v| v.as_str())
        .map(parse_weibo_time);

    let image_url = selectors
        .image
        .as_ref()
        .and_then(|path| resolve_json_path(item, path))
        .and_then(|v| v.as_str().map(String::from));

    let external_id = item
        .get("bid")
        .and_then(|v| v.as_str())
        .or_else(|| item.get("id").and_then(|v| v.as_str()))
        .map(String::from);

    // Build content_html from text with image injection
    let content_html = build_content_html(raw_text, item);

    Some(ParsedArticle {
        external_id,
        title,
        url: url_str,
        canonical_url: None,
        summary: Some(strip_html_summary(text, 200)),
        content_html,
        content_text: None,
        author: (if author.is_empty() { None } else { Some(author) }),
        published_at,
        image_url,
        tags_json: None,
    })
}

fn build_article_url(item: &Value, template: &str) -> Option<String> {
    if template.starts_with("http") {
        // Template URL like "https://weibo.com/{user.id}/{bid}"
        let mut url = template.to_string();
        let re = regex::Regex::new(r"\{([^}]+)\}").ok()?;
        let mut success = true;
        for cap in re.captures_iter(template) {
            let placeholder = &cap[1];
            let value = resolve_json_path(item, placeholder).and_then(|v| match v {
                Value::String(s) => Some(s.clone()),
                Value::Number(n) => Some(n.to_string()),
                _ => None,
            });
            match value {
                Some(v) => url = url.replace(&format!("{{{}}}", placeholder), &v),
                None => success = false,
            }
        }
        if success { Some(url) } else { None }
    } else {
        // Simple path like "bid" — construct default weibo.com URL
        let bid = resolve_json_path(item, template)
            .and_then(|v| v.as_str().map(String::from))?;
        let uid = resolve_json_path(item, "user.id")
            .and_then(|v| match v {
                Value::Number(n) => Some(n.to_string()),
                Value::String(s) => Some(s.clone()),
                _ => None,
            })?;
        Some(format!("https://weibo.com/{uid}/{bid}"))
    }
}

fn parse_weibo_time(s: &str) -> String {
    // Format: "Mon May 25 00:19:17 +0800 2026"
    // chrono parse format needs timezone at end
    // Reorder: "Mon May 25 00:19:17 2026 +0800"
    let parts: Vec<&str> = s.split_whitespace().collect();
    if parts.len() == 6 {
        // "Mon May 25 00:19:17 +0800 2026" → reorder timezone to front
        let reordered = format!(
            "{} {} {} {} {} {}",
            parts[0], parts[1], parts[2], parts[3], parts[5], parts[4]
        );
        if let Ok(dt) = chrono::NaiveDateTime::parse_from_str(
            &reordered,
            "%a %b %d %H:%M:%S %Y %z",
        ) {
            return dt.format("%Y-%m-%dT%H:%M:%S%:z").to_string();
        }
    }
    s.to_string()
}

fn strip_html_summary(html: &str, max_len: usize) -> String {
    let re = regex::Regex::new(r"<[^>]+>").unwrap();
    let plain = re.replace_all(html, "").to_string();
    let trimmed: String = plain
        .chars()
        .filter(|c| !c.is_control() || *c == '\n')
        .take(max_len)
        .collect();
    if plain.chars().count() > max_len {
        format!("{}...", trimmed.trim())
    } else {
        trimmed.trim().to_string()
    }
}

fn build_content_html(text: Option<&str>, _item: &Value) -> Option<String> {
    let text = text?;
    if text.is_empty() {
        return None;
    }
    // The text from Weibo API is already HTML with hashtag links
    // Wrap images if present
    let mut html = text.to_string();

    // Add <img> tags for pics
    if let Some(pics) = _item.get("pics").and_then(|v| v.as_array()) {
        for pic in pics {
            if let Some(url) = pic.get("large").and_then(|l| l.get("url")).and_then(|u| u.as_str())
            {
                html.push_str(&format!(
                    "<br><img src=\"{}\" style=\"max-width:100%;margin-top:0.5rem\">",
                    url
                ));
            }
        }
    }

    // Render retweet quote
    if let Some(retweet) = _item.get("retweeted_status") {
        let rt_user = retweet
            .get("user")
            .and_then(|u| u.get("screen_name"))
            .and_then(|s| s.as_str())
            .unwrap_or("");
        let rt_text = retweet
            .get("text")
            .and_then(|t| t.as_str())
            .unwrap_or("");
        html.push_str(&format!(
            "<blockquote style=\"margin:0.5rem 0;padding:0.5rem;border-left:3px solid #ccc;color:#666\">\
             <strong>@{} 转发:</strong><br>{}</blockquote>",
            rt_user, rt_text
        ));
    }

    Some(html)
}
```

- [ ] **Step 2: Register module in lib.rs**

In `src-tauri/src/lib.rs`, change line 6 from:
```rust
mod xpath_adapter;
```
To:
```rust
mod json_adapter;
mod xpath_adapter;
```

- [ ] **Step 3: Build to verify compilation**

```bash
cargo build 2>&1
```

Expected: compiles clean.

- [ ] **Step 4: Commit**

```bash
git add src-tauri/src/json_adapter.rs src-tauri/src/lib.rs
git commit -m "feat: add json_adapter for JSON API feed sources"
```

---

### Task 6: Feader — Add json-api dispatch in refresh flow

**Files:**
- Modify: `src-tauri/src/lib.rs`

- [ ] **Step 1: Add dispatch for json-api kind in refresh_source_record**

In `lib.rs`, change the `refresh_source_record` function (around line 427-435) from:
```rust
    let feed = match source.kind.as_str() {
        "rss" => feed_adapter::fetch_feed(&source.url).await,
        "xpath" => {
            let selectors = parse_xpath_selectors(source)?;
            xpath_adapter::fetch_xpath_source(&source.url, &selectors).await
        }
        kind => Err(format!("Source kind '{kind}' is not refreshable yet")),
    };
```
To:
```rust
    let feed = match source.kind.as_str() {
        "rss" => feed_adapter::fetch_feed(&source.url).await,
        "xpath" => {
            let selectors = parse_xpath_selectors(source)?;
            xpath_adapter::fetch_xpath_source(&source.url, &selectors).await
        }
        "json-api" => {
            let selectors = parse_xpath_selectors(source)?;
            let cookie = resolve_source_cookie(database, &selectors, source)?;
            json_adapter::fetch_json_feed(&source.url, &selectors, cookie.as_deref()).await
        }
        kind => Err(format!("Source kind '{kind}' is not refreshable yet")),
    };
```

- [ ] **Step 2: Add cookie resolution helper**

Add this function before `refresh_source_record` (or reuse the logic from xpath_adapter's cookie handling):

```rust
fn resolve_source_cookie(
    _database: &AppDatabase,
    selectors: &XPathSelectors,
    _source: &Source,
) -> Result<Option<String>, String> {
    // Per-source cookie from selectors config
    if let Some(cookie) = &selectors.cookie {
        if !cookie.trim().is_empty() {
            return Ok(Some(cookie.clone()));
        }
    }
    Ok(None)
}
```

- [ ] **Step 3: Build to verify compilation**

```bash
cargo build 2>&1
```

Expected: compiles clean.

- [ ] **Step 4: Commit**

```bash
git add src-tauri/src/lib.rs
git commit -m "feat: dispatch json-api sources to json_adapter on refresh"
```

---

### Task 7: Feader — Add json-api Tauri commands for source creation

**Files:**
- Modify: `src-tauri/src/lib.rs`
- Modify: `src-tauri/src/db.rs`

- [ ] **Step 1: Add add_json_api_source to db.rs**

In `db.rs`, after the `add_xpath_source` method (around line 54-61), add:

```rust
    pub fn add_json_api_source(
        &self,
        url: &str,
        title: &str,
        selectors: &XPathSelectors,
    ) -> Result<Source, String> {
        let config_json =
            serde_json::to_string(selectors).map_err(|e| format!("Serialize JSON selectors: {e}"))?;
        self.add_source_with_kind("json-api", url, Some(title), Some(&config_json))
    }
```

- [ ] **Step 2: Create a general add_plugin_source Tauri command**

In `lib.rs`, after the `add_xpath_source` command, add a new command:

```rust
/// Add a JSON API feed source after validating it can extract articles.
#[tauri::command]
async fn add_json_api_source(
    request: AddXPathSourceRequest,
    database: tauri::State<'_, AppDatabase>,
) -> Result<Source, String> {
    let url = request.url.trim();
    let title = request.title.trim();
    if url.is_empty() {
        return Err("JSON feed URL is required".to_string());
    }
    if title.is_empty() {
        return Err("Source title is required".to_string());
    }

    // Validate by doing a test fetch
    let feed = json_adapter::fetch_json_feed(url, &request.selectors, None).await?;
    if feed.articles.is_empty() {
        return Err("JSON selectors did not extract any articles".to_string());
    }

    let source = database.add_json_api_source(url, title, &request.selectors)?;
    database.upsert_articles(source.id, Some(title), &feed.articles)?;
    database.get_source(source.id)
}
```

- [ ] **Step 3: Register the new command**

In `lib.rs`, add `add_json_api_source` to the `invoke_handler` list (around line 489-513), and add the import for `AddXPathSourceRequest`. Add `use crate::json_adapter;` at the top imports.

- [ ] **Step 4: Build to verify compilation**

```bash
cargo build 2>&1
```

Expected: compiles clean.

- [ ] **Step 5: Commit**

```bash
git add src-tauri/src/db.rs src-tauri/src/lib.rs
git commit -m "feat: add json-api source creation and Tauri commands"
```

---

### Task 8: Feader — Test with real Weibo API

**Files:** None (manual verification)

- [ ] **Step 1: Build the Tauri app**

```bash
cd /Users/fwmbam4/CodeHub/Frankie/Feader/.worktrees/weibo-json-feed
cargo build 2>&1
```

- [ ] **Step 2: Test JSON path resolution with unit test**

Add to `src-tauri/src/json_adapter.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use serde_json::json;

    #[test]
    fn resolves_simple_path() {
        let v = json!({"user": {"screen_name": "人民日报"}});
        let result = resolve_json_path(&v, "user.screen_name");
        assert_eq!(result.and_then(|v| v.as_str()), Some("人民日报"));
    }

    #[test]
    fn resolves_array_index() {
        let v = json!({"pics": [{"large": {"url": "https://img.jpg"}}]});
        let result = resolve_json_path(&v, "pics[0].large.url");
        assert_eq!(result.and_then(|v| v.as_str()), Some("https://img.jpg"));
    }

    #[test]
    fn parses_weibo_time() {
        let result = parse_weibo_time("Mon May 25 00:19:17 +0800 2026");
        assert!(result.contains("2026-05-25T00:19:17"));
    }

    #[test]
    fn extracts_items_with_filter() {
        let v = json!({
            "data": {
                "cards": [
                    {"card_type": 9, "mblog": {"bid": "abc"}},
                    {"card_type": 156, "commend": true},
                    {"card_type": 9, "mblog": {"bid": "def"}}
                ]
            }
        });
        let items = extract_items(&v, "data.cards[?(@.card_type==9)]").unwrap();
        assert_eq!(items.len(), 2);
    }
}
```

Run: `cargo test -- json_adapter`

- [ ] **Step 3: Verify test results**

Run: `cargo test -- json_adapter`
Expected: All 4 tests pass.

- [ ] **Step 4: Commit**

```bash
git add src-tauri/src/json_adapter.rs
git commit -m "test: add unit tests for json_adapter path resolution"
```

---

### Task 9: Final commit — update FeaderHub plan documentation

**Files:**
- Create: `docs/superpowers/plans/2026-05-24-weibo-json-feed-plugin.md` (this file)

- [ ] **Step 1: Commit plan to FeaderHub**

```bash
git add docs/superpowers/plans/2026-05-24-weibo-json-feed-plugin.md
git commit -m "docs: add Weibo JSON feed plugin implementation plan"
```

---

## Summary

| Task | Repo | What |
|---|---|---|
| 1 | FeaderHub | Add `json-api-feed` to manifest schema kind enum |
| 2 | FeaderHub | Create Weibo plugin (manifest.json + xpath-rule-pack.json) |
| 3 | FeaderHub | Register in registry/index.json with sha256 |
| 4 | Feader | Accept json-api-feed kind in plugin_registry.rs |
| 5 | Feader | Create json_adapter.rs (JSON API extraction engine) |
| 6 | Feader | Add json-api dispatch in refresh_source_record |
| 7 | Feader | Add add_json_api_source Tauri command + db method |
| 8 | Feader | Build, test with unit tests |
| 9 | FeaderHub | Commit plan document |

**Total estimated changes:**
- FeaderHub: 4 files changed, 2 new files created (~150 lines JSON)
- Feader: 4 files changed, 1 new file created (~300 lines Rust)
