---
name: wechat-official-account-draft
description: "Use when publishing articles as drafts to WeChat Official Account (微信公众号). Supports two methods: API (requires AppID/AppSecret) and browser automation (requires QR scan login)."
version: 2.0.0
author: Juno Team
license: MIT
platforms: [macos, linux]
required_environment_variables:
  - name: WECHAT_APPID
    prompt: WeChat Official Account AppID
    help: Find in WeChat Developer Platform (微信开发者平台) → your app → basic info
    required_for: API method only (not needed for browser automation)
  - name: WECHAT_APPSECRET
    prompt: WeChat Official Account AppSecret
    help: Find in WeChat Developer Platform (微信开发者平台) → your app → basic settings. Reset if forgotten.
    required_for: API method only (not needed for browser automation)
metadata:
  hermes:
    tags: [wechat, official-account, draft, publishing, social-media]
    category: social-media
    related_skills: []
---

# WeChat Official Account Draft Management (微信公众号草稿管理)

## Overview

This skill enables publishing articles as drafts to a WeChat Official Account (公众号). It supports **two methods**:

| Method | Requirements | Best for |
|--------|-------------|----------|
| **A: API** | AppID + AppSecret + IP whitelist | Technical users, automation pipelines, batch operations |
| **B: Browser Automation** | Browser + QR code scan | Non-technical users, no developer setup needed |

Pick Method A when you have API credentials. Fall back to Method B when they're not available.

## When to Use

- You want to create, edit, or publish articles on a WeChat Official Account programmatically
- You need to upload images for use in WeChat article content
- You're automating a content publishing pipeline to 微信公众号
- You need to list, inspect, update, or delete existing drafts
- You want to submit a draft for publishing (发布)

**Don't use for:**
- Sending WeChat customer service messages (use 客服消息 API instead)
- WeChat Mini Program development (different API system)
- WeChat web page JS-SDK features

## Method Selection

Check which method is available:

```python
import os
if os.getenv("WECHAT_APPID") and os.getenv("WECHAT_APPSECRET"):
    print("Method A (API) is available — preferred")
else:
    print("Method B (Browser Automation) — fallback")
```

## Prerequisites

**For Method A (API):**
- A WeChat Official Account (公众号) with developer mode enabled
- **AppID** and **AppSecret** from the WeChat Developer Platform (微信开发者平台 → 开发 → 基本配置)
- The calling server's IP must be added to the IP whitelist (微信开发者平台 → 开发管理 → IP白名单)
- `curl` available on the system

**For Method B (Browser Automation):**
- A WeChat Official Account login credential (微信公众平台账号密码, or phone bound to the account)
- The `browser` toolset must be available (browser_navigate, browser_click, browser_type, browser_vision)
- User must be able to scan a QR code with their WeChat app

## Environment Variables

Set these in `~/.hermes/.env` or export them before using the skill:

| Variable | Description | Required For |
|----------|-------------|-------------|
| `WECHAT_APPID` | Official Account AppID | Method A only |
| `WECHAT_APPSECRET` | Official Account AppSecret | Method A only |

If neither is set, the skill will use Method B (browser automation).

## Available Scripts

This skill bundles reusable scripts in the `scripts/` directory. Each script outputs JSON and supports `--help`.

| Script | Purpose | Example |
|--------|---------|---------|
| `scripts/wechat-get-token` | Get access token (API method) | `--appid ID --secret KEY` |
| `scripts/wechat-upload-image` | Upload image via API | `--token T --file img.jpg --type material` |
| `scripts/wechat-create-draft` | Create draft via API | `--token T --title "X" --content "<p>Y</p>"` |
| `scripts/wechat-list-drafts` | List / get / count drafts via API | `--token T list` |
| `scripts/wechat-delete-draft` | Delete a draft via API | `--token T --media-id M` |
| `scripts/wechat-publish-draft` | Submit draft for publishing via API | `--token T --media-id M` |
| `scripts/wechat-browser-draft` | Create draft via browser automation (QR login) | `--title "X" --content "<p>Y</p>"` |

All scripts output JSON (`{"success": bool, ...}`) and support `--help`.

**API scripts** (first 6) require `WECHAT_APPID` + `WECHAT_APPSECRET`. Run them from the skill root:
```bash
./scripts/wechat-get-token --appid "$WECHAT_APPID" --secret "$WECHAT_APPSECRET"
```

**Browser script** (`wechat-browser-draft`) requires Playwright and doesn't need API credentials:
```bash
pip install playwright && playwright install chromium
./scripts/wechat-browser-draft --title "My Article" --content "<p>Hello</p>"
```

## Procedure

The standard workflow follows these steps:

### Step 1: Get Access Token

Obtain the API credential. Valid for 7200 seconds — cache and reuse it.

```
GET https://api.weixin.qq.com/cgi-bin/token?appid=APPID&secret=APPSECRET&grant_type=client_credential
```

**Response:**
```json
{"access_token": "ACCESS_TOKEN", "expires_in": 7200}
```

**Implementation:**
```bash
TOKEN_RESP=$(curl -s "https://api.weixin.qq.com/cgi-bin/token?appid=$APPID&secret=$APPSECRET&grant_type=client_credential")
ACCESS_TOKEN=$(echo "$TOKEN_RESP" | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")
```

### Step 2: Upload Cover Image (if needed)

Upload a cover image as a **permanent material** to get a `thumb_media_id`. Required for news-type (图文消息) drafts.

```
POST https://api.weixin.qq.com/cgi-bin/material/add_material?access_token=ACCESS_TOKEN
```

Form data: `media=@/path/to/image.jpg`, `type=image`

**Response:**
```json
{"media_id": "pEG...", "url": "http://mmbiz.qpic.cn/..."}
```

### Step 3: Upload Content Images (if needed)

Upload images that will appear inside the article body. External image URLs are filtered out by WeChat.

```
POST https://api.weixin.qq.com/cgi-bin/media/uploadimg?access_token=ACCESS_TOKEN
```

Form data: `media=@/path/to/image.jpg`

**Response:**
```json
{"url": "https://mmbiz.qpic.cn/..."}
```

Use the returned URL in the article HTML: `<img src="https://mmbiz.qpic.cn/..." />`

### Step 4: Create Draft

Create the draft article in the draft box.

```
POST https://api.weixin.qq.com/cgi-bin/draft/add?access_token=ACCESS_TOKEN
```

**Request body:**
```json
{
  "articles": [{
    "title": "Article Title (max 32 chars)",
    "author": "Author Name",
    "digest": "Summary (max 128 chars)",
    "content": "<p>HTML content with uploaded image URLs</p>",
    "content_source_url": "https://...",
    "thumb_media_id": "PERMANENT_MEDIA_ID_FROM_STEP_2",
    "need_open_comment": 1,
    "only_fans_can_comment": 0
  }]
}
```

**Response:**
```json
{"media_id": "MEDIA_ID"}
```

### Step 5: Verify Draft

Verify the draft was created correctly.

```
POST https://api.weixin.qq.com/cgi-bin/draft/get?access_token=ACCESS_TOKEN
Body: {"media_id": "MEDIA_ID"}
```

### Step 6 (Optional): Submit for Publishing

Submit the draft for publishing. The draft is automatically removed from the draft box after submission.

```
POST https://api.weixin.qq.com/cgi-bin/freepublish/submit?access_token=***
Body: {"media_id": "MEDIA_ID"}
```

**Response:**
```json
{"publish_id": "PUBLISH_ID"}
```

---

## Method B: Browser Automation

Use this method when the user does not have WECHAT_APPID / WECHAT_APPSECRET set. This approach uses the `wechat-browser-draft` script with Playwright to automate mp.weixin.qq.com.

### Key Design Decisions

1. **QR code login only** — no account/password support. The browser shows a QR code; the user scans it with their WeChat app.
2. **Session timeout detection** — every step checks if the WeChat session has expired (detects "登录超时" / "请重新登录" text). If timed out, it clicks the login link, waits for a new QR code, and asks the user to re-scan.
3. **Browser stays open** — the login session is bound to the browser process. The script keeps the browser alive so the session isn't lost.

### Usage

```bash
pip3 install playwright
python3 -m playwright install chromium

cd wechat-official-account-draft/
./scripts/wechat-browser-draft --title "My Article" --content "<p>Hello</p>"
```

### Step-by-step flow

1. Script launches Chromium browser window
2. Navigates to `https://mp.weixin.qq.com/`
3. QR code appears — **you scan it with your WeChat app**
4. After login, script navigates to the material management page
5. Clicks "新建" (New) to open the article editor
6. Fills in title, content, author, digest
7. Uploads cover image if `--cover` is provided
8. Clicks "保存" (Save) to save as draft
9. Keeps browser open so you can verify the result
10. Close the browser window manually when done

### Session Timeout Recovery

If the script detects a timeout page at any step, it will:

```
⚠️  Session timed out! Re-login required.
ℹ️  Please scan the QR code with your WeChat app.
```

Click the login link → new QR code appears → scan again → script continues from where it paused.

### Important

- **Don't close the browser** while the script is running — that kills the session.
- The QR code expires after ~2 minutes. If you miss it, the script will detect the timeout and show a new QR code.
- If the script can't find a form field (e.g., title input), it saves a debug screenshot to `/tmp/wechat_debug_title.png`.

### Markdown Support

Both `wechat-browser-draft` and `wechat-create-draft` accept **Markdown content** and automatically convert it to HTML:

```bash
# Markdown is auto-detected and converted
./scripts/wechat-browser-draft \
  --title "Markdown测试" \
  --content "# 标题

支持 **加粗**、*斜体*、\`代码\` 和 [链接](https://example.com)

- 无序列表项1
- 无序列表项2

1. 有序列表项1
2. 有序列表项2"
```

**Detection logic:**
- If `--content` contains HTML tags (`<p>`, `<h1>`, etc.), it's used as-is
- If it looks like plain text / markdown, it's converted to HTML
- Conversion priority: `python-markdown` library → `mistune` → built-in regex fallback
- Supports: `#`/`##`/`###` headers, `**bold**`, `*italic*`, `` `code` ``, `[links](url)`, `- lists`, `1. ordered lists`

## Switching Between Methods

Use this pattern to pick the right method:

```python
import os

if os.getenv("WECHAT_APPID") and os.getenv("WECHAT_APPSECRET"):
    print("Using Method A (API)")
    # Proceed with API calls
else:
    print("Using Method B (Browser Automation)")
    # Proceed with browser automation
```

## API Reference

### GET Access Token

Gets the API credential required for all subsequent calls. Token is valid for 7200 seconds.

```
GET https://api.weixin.qq.com/cgi-bin/token?appid=APPID&secret=APPSECRET&grant_type=client_credential
```

**Response:**
```json
{
  "access_token": "ACCESS_TOKEN",
  "expires_in": 7200
}
```

**Implementation pattern:**
```bash
# Get access token
TOKEN_RESP=$(curl -s "https://api.weixin.qq.com/cgi-bin/token?appid=$APPID&secret=$APPSECRET&grant_type=client_credential")
ACCESS_TOKEN=$(echo "$TOKEN_RESP" | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")
```

> **Important:** Cache the token and reuse it within the 7200s window. Do not request a new one for every API call.

### 2. Upload Image for Article Content

Uploads an image that will be used inside the article body. The returned URL must be used in the article content — external image URLs will be filtered out.

```
POST https://api.weixin.qq.com/cgi-bin/media/uploadimg?access_token=ACCESS_TOKEN
```

**Request:** multipart/form-data with the image file as field `media`.

**Response:**
```json
{
  "url": "https://mmbiz.qpic.cn/..."
}
```

**Implementation pattern:**
```bash
UPLOAD_RESP=$(curl -s -F "media=@/path/to/image.jpg" "https://api.weixin.qq.com/cgi-bin/media/uploadimg?access_token=$ACCESS_TOKEN")
IMG_URL=$(echo "$UPLOAD_RESP" | python3 -c "import sys,json; print(json.load(sys.stdin)['url'])")
```

> **Note:** The uploaded image URL is only valid for use in articles from the same account. The image must be uploaded before creating the draft.

### 3. Create Draft (新增草稿)

Creates a new draft article in the draft box.

```
POST https://api.weixin.qq.com/cgi-bin/draft/add?access_token=ACCESS_TOKEN
```

**Request Body:**
```json
{
  "articles": [
    {
      "title": "Article Title (max 32 chars)",
      "author": "Author Name (max 16 chars)",
      "digest": "Article summary (max 128 chars, defaults to first 54 chars of content)",
      "content": "<p>Article HTML content. Must use image URLs from the uploadimg API. External URLs are filtered.</p>",
      "content_source_url": "https://... (original article URL for 'read more' link)",
      "thumb_media_id": "PERMANENT_MEDIA_ID (required for news articles)",
      "need_open_comment": 0,
      "only_fans_can_comment": 0
    }
  ]
}
```

**Response:**
```json
{
  "media_id": "MEDIA_ID"
}
```

**Parameter Details:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| article_type | string | No | "news" (图文消息, default) or "newspic" (图片消息) |
| title | string | **Yes** | Max 32 characters |
| author | string | No | Max 16 characters |
| digest | string | No | Max 128 characters |
| content | string | **Yes** | HTML content, < 20K chars, < 1MB, must use uploaded image URLs |
| content_source_url | string | No | "Read more" link URL |
| thumb_media_id | string | No* | **Required for news articles** — permanent media ID for cover image |
| need_open_comment | number | No | 0=off(default), 1=on |
| only_fans_can_comment | number | No | 0=everyone(default), 1=fans only |
| pic_crop_235_1 | string | No | Crop coordinates for 2.35:1 cover (format: "x1_y1_x2_y2") |
| pic_crop_1_1 | string | No | Crop coordinates for 1:1 cover |
| image_info | object | No | For newpic (图片消息) type only |
| product_info | object | No | Product card info |

**Implementation pattern:**
```bash
# Create draft
DRAFT_RESP=$(curl -s -X POST \
  "https://api.weixin.qq.com/cgi-bin/draft/add?access_token=$ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "articles": [{
      "title": "文章标题",
      "author": "作者名",
      "content": "<p>文章内容</p>",
      "thumb_media_id": "PERMANENT_MEDIA_ID"
    }]
  }')
MEDIA_ID=$(echo "$DRAFT_RESP" | python3 -c "import sys,json; print(json.load(sys.stdin)['media_id'])")
```

### 4. Get Draft List (获取草稿列表)

Retrieves a paginated list of all drafts.

```
POST https://api.weixin.qq.com/cgi-bin/draft/batchget?access_token=ACCESS_TOKEN
```

**Request Body:**
```json
{
  "offset": 0,
  "count": 10,
  "no_content": 0
}
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| offset | number | Yes | Starting position (0-based) |
| count | number | Yes | Number to retrieve (1-20) |
| no_content | number | No | 0=include content(default), 1=omit content |

**Response:**
```json
{
  "total_count": 5,
  "item_count": 2,
  "items": [
    {
      "media_id": "MEDIA_ID",
      "content": {
        "news_item": [
          {
            "title": "...",
            "thumb_media_id": "...",
            "show_cover_pic": 0,
            "author": "...",
            "digest": "...",
            "content": "...",
            "url": "https://mp.weixin.qq.com/...",
            "content_source_url": "...",
            "need_open_comment": 0,
            "only_fans_can_comment": 0
          }
        ]
      },
      "update_time": 1234567890
    }
  ]
}
```

### 5. Get Draft Details (获取草稿详情)

Retrieves a specific draft by media_id.

```
POST https://api.weixin.qq.com/cgi-bin/draft/get?access_token=ACCESS_TOKEN
```

**Request Body:**
```json
{
  "media_id": "MEDIA_ID"
}
```

**Response:** Same format as individual items in the batchget response.

### 6. Update Draft (更新草稿)

Modifies an existing draft.

```
POST https://api.weixin.qq.com/cgi-bin/draft/update?access_token=ACCESS_TOKEN
```

**Request Body:**
```json
{
  "media_id": "MEDIA_ID",
  "articles": [
    {
      "title": "Updated Title",
      "content": "<p>Updated content</p>",
      "thumb_media_id": "PERMANENT_MEDIA_ID"
    }
  ],
  "index": 0
}
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| media_id | string | **Yes** | Draft media_id to update |
| articles | array | **Yes** | Updated article(s) — same structure as draft_add |
| index | number | No | Article index in multi-article drafts (0-based) |

### 7. Delete Draft (删除草稿)

Deletes a draft from the draft box.

```
POST https://api.weixin.qq.com/cgi-bin/draft/delete?access_token=ACCESS_TOKEN
```

**Request Body:**
```json
{
  "media_id": "MEDIA_ID"
}
```

### 8. Get Draft Count (获取草稿总数)

Returns the total count of drafts.

```
POST https://api.weixin.qq.com/cgi-bin/draft/count?access_token=ACCESS_TOKEN
```

**Response:**
```json
{
  "total_count": 5
}
```

### 9. Submit Draft for Publishing (发布草稿)

Submits a draft for publishing. After submission, the draft is removed from the draft box.

```
POST https://api.weixin.qq.com/cgi-bin/freepublish/submit?access_token=ACCESS_TOKEN
```

**Request Body:**
```json
{
  "media_id": "MEDIA_ID"
}
```

**Response:**
```json
{
  "publish_id": "PUBLISH_ID"
}
```

The publish_id can be used to check the publish status via the publish status query API.

### 10. Check Publish Status (发布状态查询)

```
POST https://api.weixin.qq.com/cgi-bin/freepublish/get?access_token=ACCESS_TOKEN
```

**Request Body:**
```json
{
  "publish_id": "PUBLISH_ID"
}
```

## Common Workflow: Create and Publish an Article

```bash
#!/bin/bash
# Full workflow: upload image → create draft → submit for publishing

APPID="${WECHAT_APPID}"
APPSECRET="${WECHAT_APPSECRET}"
IMAGE_PATH="$1"  # Image file path (optional)
TITLE="$2"       # Article title
CONTENT="$3"     # Article HTML content

# Step 1: Get access token
TOKEN_RESP=$(curl -s "https://api.weixin.qq.com/cgi-bin/token?appid=$APPID&secret=$APPSECRET&grant_type=client_credential")
ACCESS_TOKEN=$(echo "$TOKEN_RESP" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('access_token',''))")
if [ -z "$ACCESS_TOKEN" ]; then
  echo "ERROR: Failed to get access token"
  echo "$TOKEN_RESP"
  exit 1
fi

# Step 2 (optional): Upload image if provided
IMG_URL=""
if [ -n "$IMAGE_PATH" ] && [ -f "$IMAGE_PATH" ]; then
  UPLOAD_RESP=$(curl -s -F "media=@$IMAGE_PATH" \
    "https://api.weixin.qq.com/cgi-bin/media/uploadimg?access_token=$ACCESS_TOKEN")
  IMG_URL=$(echo "$UPLOAD_RESP" | python3 -c "import sys,json; print(json.load(sys.stdin).get('url',''))")
  echo "Image uploaded: $IMG_URL"
fi

# Step 3: Create draft
DRAFT_RESP=$(curl -s -X POST \
  "https://api.weixin.qq.com/cgi-bin/draft/add?access_token=$ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d "$(python3 -c "
import json
article = {
    'articles': [{
        'title': '$TITLE',
        'content': '$CONTENT',
        'need_open_comment': 1,
        'only_fans_can_comment': 0
    }]
}
print(json.dumps(article, ensure_ascii=False))
")")
MEDIA_ID=$(echo "$DRAFT_RESP" | python3 -c "import sys,json; print(json.load(sys.stdin).get('media_id',''))")
echo "Draft created: $MEDIA_ID"
```

## Common Pitfalls

**Method A (API):**

1. **thumb_media_id is required for news articles (图文消息).** You MUST upload a permanent cover image first via `/cgi-bin/material/add_material` (type=image) to get a permanent media_id. The `uploadimg` endpoint returns a URL, not a media_id — those are different endpoints for different purposes.

2. **Content image URLs must come from uploadimg API.** External image URLs (e.g. from GitHub, CDN) are silently filtered out. Always upload images first via `/cgi-bin/media/uploadimg` and use the returned `url` in the article HTML.

3. **IP whitelist required.** The WeChat API requires the calling server's IP to be added to the IP whitelist in the WeChat Developer Platform (微信开发者平台 → 开发管理 → IP白名单). Without this, you'll get error 40164.

4. **Access token expires after 7200s.** Cache and reuse the token. Requesting too frequently may hit rate limits.

5. **Error codes are important.** Always check `errcode` in responses — a non-zero errcode indicates failure even with HTTP 200.

**Method B (Browser Automation):**

6. **Browser session must stay alive.** Login state is tied to the browser process. If the browser is closed, the user must re-scan the QR code. The script intentionally keeps the browser open.

7. **QR code has a ~2 minute expiry.** If you don't scan in time, the script detects the timeout via `is_session_timeout()`, clicks the login button, and shows a fresh QR code. Just scan again.

8. **WeChat UI changes frequently.** The editor may use different HTML structures over time. The script uses JavaScript injection (`page.evaluate`) as a fallback to set title/content when DOM selectors fail. Debug screenshots are saved to `/tmp/wechat_debug_*.png`.

9. **Title length limit (32 chars).** Same as the API. The web editor may silently truncate titles over 32 Chinese characters.

10. **No `required_environment_variables` needed for this method.** Fall back to browser automation when `WECHAT_APPID` / `WECHAT_APPSECRET` are not set.

## Reference: Common Error Codes

| Error Code | Description |
|------------|-------------|
| 0 | Success |
| -1 | System error |
| 40001 | Invalid access token |
| 40007 | Invalid media_id |
| 41001 | Missing access_token |
| 42001 | Access token expired |
| 45009 | API call quota exceeded |
| 53404 | Account has restricted带货 ability |
| 53405 | Product info error |
| 53406 | Need to enable带货能力 |

## Verification

**Method A (API) checks:**
- [ ] Access token is obtained successfully (non-empty, valid format)
- [ ] Image upload returns a valid URL starting with `https://mmbiz.qpic.cn/`
- [ ] Draft creation API returns a non-empty `media_id`
- [ ] Draft can be retrieved via `draft/get` with the returned media_id
- [ ] Draft appears in `draft/batchget` list
- [ ] Publish submission returns a non-empty `publish_id`

**Method B (Browser Automation) checks:**
- [ ] Browser launched and QR code displayed
- [ ] User scanned QR code and login succeeded
- [ ] Script navigated to material management page
- [ ] "新建" button clicked successfully
- [ ] Title was filled in correctly
- [ ] Content was set in the editor
- [ ] Save button clicked and no error returned
- [ ] Draft appears in 草稿箱 (Draft box) on mp.weixin.qq.com
