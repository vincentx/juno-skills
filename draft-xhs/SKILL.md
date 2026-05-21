---
name: draft-xhs
description: "Use when creating image-text drafts (图文笔记草稿) on Xiaohongshu (小红书). Uploads images, fills title/content, then publishes as timed draft (定时发布) to sync to mobile for later review."
version: 1.3.0
author: Juno Team
license: MIT
platforms: [macos, linux]
metadata:
  hermes:
    tags: [xiaohongshu, xhs, image, draft, social-media]
    category: social-media
    related_skills: [wechat-official-account-draft]
---

# Xiaohongshu Scheduled Publish (小红书定时发布)

## Overview

Publishes image-text notes on Xiaohongshu via browser automation. Uploads images, fills title/content, enables **定时发布** (scheduled publish, max 14 days), and confirms.

**Method: Browser Automation only** — no public API.

## Flow

1. **Login** — `creator.xiaohongshu.com/login` → QR code → scan
2. **Navigate** → `creator.xiaohongshu.com/publish/publish?from=tab_switch&target=image`
3. **Upload** — filechooser event
4. **Title** — `<input class="d-text" placeholder="填写标题会有更多赞哦">` (use `press_sequentially`)
5. **Content** — `<div class="tiptap ProseMirror">` (ClipboardEvent paste)
6. **Scheduled publish** — click `.post-time-wrapper .d-switch-simulator` toggle → set date 14 days later via JS
7. **Publish** — CDP to find `<button class="ce-btn bg-red">` inside closed shadow DOM

## Key Technical Details

| Component | DOM | Approach |
|-----------|-----|----------|
| Title input | `<input class="d-text" placeholder="填写标题会有更多赞哦">` | `press_sequentially()` |
| Content editor | `<div class="tiptap ProseMirror" contenteditable="true">` | ClipboardEvent paste with DataTransfer |
| 定时发布 toggle | `<span class="d-switch-simulator unchecked">` in `.post-time-wrapper` | `.click()` on the span |
| Date picker | `<input class="d-text">` in `.d-datepicker` | JS native setter + `input` event |
| Publish button | `<button class="ce-btn bg-red">` in `<xhs-publish-btn>` shadow DOM | CDP: `querySelector` on shadow root |

## Script

```bash
./scripts/xhs-browser-publish --title "标题" --content "正文" --images-dir ./photos/ --debug
```

## Common Pitfalls

1. **Creator subdomain needs separate login** from main site
2. **Geolocation**: add `permissions=["geolocation"]` to browser context
3. **Vue controlled inputs**: `page.fill()` might not trigger state; use `press_sequentially`
4. **TipTap/ProseMirror**: `execCommand` ignored; use ClipboardEvent paste with DataTransfer
5. **Closed Shadow DOM** (`shadowrootmode="closed"`): CDP required to access buttons
6. **`page.evaluate` arrow functions**: `() => { arguments[0] }` fails; use `function() { arguments[0] }`
7. **File upload**: use `expect_event("filechooser")` pattern
