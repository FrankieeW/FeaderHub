# Weibo JSON Feed Plugin Design

## Overview

Add Weibo (微博) user feed support to Feader ecosystem via a new `json-api-feed` plugin kind. Unlike `static-xpath-rule-pack` plugins that scrape HTML with XPath, this plugin consumes JSON APIs directly — Weibo's m.weibo.cn API returns structured JSON that XPath cannot access (the HTML page is a Vue SPA skeleton).

## Motivation

- **Weibo HTML is a SPA**: m.weibo.cn pages are Vue apps — server returns placeholder skeletons, real content is rendered client-side. XPath extraction is impossible.
- **JSON API is real**: `m.weibo.cn/api/container/getIndex` returns structured mblog data with 30+ fields when proper cookies are present.
- **Reusable pattern**: Many modern sites use JSON APIs. A `json-api-feed` adapter benefits all future JSON-source plugins (Douyin, Xiaohongshu, Jike, etc).

## Component Changes

### FeaderHub (plugin definition)
- New plugin: `plugins/official.weibo-user-feed.xpath/`
- Schema: reuse existing `xpath-rule-pack.json` structure where field paths are JSON paths instead of XPath
- Manifest `kind`: `"json-api-feed"` (new value)

### Feader App (runtime engine)
- `plugin_registry.rs`: accept `json-api-feed` kind alongside `static-xpath-rule-pack`
- `models.rs`: reuse `XPathRulePack` / `XPathSelectors` types (path semantics change)
- `json_adapter.rs`: **new file** — JSON API extraction engine
- `lib.rs`: add `"json-api"` branch to `refresh_source_record` dispatch

## Plugin Configuration

### manifest.json
```json
{
  "schemaVersion": "feader-plugin/v1",
  "id": "official.weibo-user-feed.xpath",
  "name": "Weibo User Feed",
  "version": "0.1.0",
  "kind": "json-api-feed",
  "feaderApiVersion": "json-api-feed/v1",
  "entry": "xpath-rule-pack.json",
  "permissions": {
    "network": ["m.weibo.cn", "sinaimg.cn", "weibo.com"],
    "credentials": ["cookie"],
    "execution": "none"
  }
}
```

### xpath-rule-pack.json (reused structure, JSON path semantics)
```json
{
  "schemaVersion": "xpath-rule-pack/v1",
  "id": "official.weibo-user-feed.xpath",
  "name": "Weibo User Feed",
  "version": "0.1.0",
  "parameters": {
    "urlTemplate": "https://m.weibo.cn/api/container/getIndex?type=uid&value={uid}&containerid={cid}",
    "profileUrl": "https://m.weibo.cn/api/container/getIndex?type=uid&value={uid}",
    "defaults": { "maxItems": 40, "maxPages": 5 }
  },
  "candidates": [{
    "id": "weibo-user-feed",
    "pageType": "social-feed",
    "priority": 90,
    "detect": ["mblog", "card_type"],
    "selectors": {
      "items": "data.cards[?(@.card_type==9)]",
      "title": "user.screen_name",
      "url": "bid",
      "summary": "text",
      "publishedAt": "created_at",
      "author": "user.screen_name",
      "image": "pics[0].large.url",
      "content": "text",
      "nextPage": "data.cardlistInfo.since_id",
      "customFields": [
        { "key": "reposts",   "path": "reposts_count" },
        { "key": "comments",  "path": "comments_count" },
        { "key": "likes",     "path": "attitudes_count" },
        { "key": "source",    "path": "source" }
      ]
    }
  }]
}
```

## Data Flow

```
User configures source (uid, cookie)
  → Feader loads manifest + rule pack from FeaderHub
  → json_adapter constructs API URL from urlTemplate
  → GET m.weibo.cn/api/container/getIndex?type=uid&value={uid}&containerid={cid}
     (with cookie, mobile UA, MWeibo-Pwa headers)
  → Parse JSON response
  → Extract cards where card_type == 9 (status posts)
  → For each mblog: apply JSON paths to extract fields
  → URL generation: https://weibo.com/{user.id}/{bid}
  → Time parsing: Weibo format → ISO 8601
  → Pagination: since_id cursor → append since_id={value} to next request
  → Return ParsedFeed to Feader storage pipeline
```

## JSON Path Syntax

Minimal implementation — dot notation for objects, bracket for arrays:

| Path | Input | Result |
|---|---|---|
| `user.screen_name` | `{"user":{"screen_name":"人民日报"}}` | `"人民日报"` |
| `pics[0].large.url` | `{"pics":[{"large":{"url":"https://..."}}]}` | `"https://..."` |
| `data.cards` | `{"data":{"cards":[...]}}` | `[...]` |

Special paths:
- `?(@.key==value)` — filter array by field match
- `{uid}`, `{bid}` — template variables in URL construction

## Weibo API Details

### Endpoints Used
1. **User profile**: `GET /api/container/getIndex?type=uid&value={uid}`
   - Returns `tabsInfo.tabs[].containerid` for finding the posts tab
2. **User posts**: `GET /api/container/getIndex?type=uid&value={uid}&containerid={cid}`
   - `cid` is typically `107603{uid}` for the "微博" tab
3. **Detail** (optional): `GET /statuses/show?id={bid}`
   - For expanding `isLongText` posts (future enhancement)

### Required Headers
```
User-Agent: Mozilla/5.0 (iPhone; CPU iPhone OS 16_0 ...) Safari/604.1
MWeibo-Pwa: 1
X-Requested-With: XMLHttpRequest
Referer: https://m.weibo.cn/u/{uid}
```

### Authentication
- Cookie-based: requires `SUB` cookie (visitor or logged-in)
- Without valid cookie: API returns `ok: -100`
- Cookie is per-source in Feader's credential system

### Pagination
- Cursor-based: `since_id` from `data.cardlistInfo.since_id`
- Append `&since_id={value}` to next request
- Stop when `since_id` is 0 or missing

## Mblog Field Mapping

| JSON field | Feader field | Notes |
|---|---|---|
| `user.screen_name` | title, author | Post attribution |
| `bid` | url | Template: `https://weibo.com/{user.id}/{bid}` |
| `text` | summary, content | HTML with hashtag links |
| `created_at` | publishedAt | Format: "Mon May 25 00:19:17 +0800 2026" |
| `pics[0].large.url` | image | First image thumbnail |
| `reposts_count` | customFields.reposts | |
| `comments_count` | customFields.comments | |
| `attitudes_count` | customFields.likes | |
| `source` | customFields.source | Posting device info |
| `retweeted_status` | (future) | Retweet chain rendering |

## Feader Implementation (json_adapter.rs)

Rough module structure:
```
json_adapter.rs
  pub fn parse_json_feed(source: &Source) -> Result<ParsedFeed>
  fn construct_api_url(template: &str, params: &HashMap) -> String
  fn resolve_json_path(value: &Value, path: &str) -> Option<Value>
  fn filter_array_by_field(arr: &[Value], key: &str, val: &str) -> Vec<Value>
  fn parse_weibo_time(s: &str) -> Option<DateTime<Utc>>
```

Dependencies: `serde_json::Value`, existing `reqwest` HTTP client, `cookie_header_value`.

## Scope

**In scope (first version):**
- Weibo user feed (list posts by user UID)
- JSON path field extraction
- since_id cursor pagination
- Cookie authentication via Feader credential system
- Custom fields for social metrics

**Out of scope (future versions):**
- Long text expansion (isLongText → statuses/show)
- Retweet chain rendering
- Article expansion
- Hot comment fetching
- Hot search / super topic / keyword search
- Friends timeline (requires stronger auth)

## Safety

- Network requests restricted to declared `permissions.network` domains only
- Cookie injection managed by Feader credential system (per-plugin credentials table)
- No code execution — purely declarative JSON path mapping
- Response size bounded by existing Feader limits
