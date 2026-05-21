# Juno Skills

Hermes Agent 可复用技能集合。

## Skills

### wechat-official-account-draft

微信公众号草稿管理。支持两种方式：
- **API 方式**：需要 AppID + AppSecret（开发者模式）
- **浏览器自动化**：扫码登录 Playwright 自动操作

功能包括：
- 新建/编辑/删除草稿
- Markdown 自动转 HTML
- 上传封面图片
- 提交发布

```bash
cd wechat-official-account-draft
# API 方式
./scripts/wechat-get-token --appid $APPID --secret $APPSECRET
./scripts/wechat-create-draft --token $TOKEN --title "标题" --content "# Markdown"

# 浏览器方式
./scripts/wechat-browser-draft --title "标题" --content "# Markdown" --author "Juno"
```

详细文档见各 skill 目录下的 `SKILL.md`。
