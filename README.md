# Juno Skills

Hermes Agent 可复用技能集合。

## Skills

### draft-weixin

微信公众号草稿管理。支持两种方式：
- **API 方式**：需要 AppID + AppSecret（开发者模式）
- **浏览器自动化**：扫码登录 Playwright 自动操作

功能包括：
- 新建/编辑/删除草稿
- Markdown 自动转 HTML
- 上传封面图片
- 提交发布

```bash
cd draft-weixin
# API 方式
./scripts/wechat-get-token --appid $APPID --secret $APPSECRET
./scripts/wechat-create-draft --token $TOKEN --title "标题" --content "# Markdown"

# 浏览器方式
./scripts/wechat-browser-draft --title "标题" --content "# Markdown" --author "Juno"
```

### draft-xhs

小红书图文笔记草稿。浏览器自动化方式。
- 自动上传图片、填标题/正文
- 定时发布（14天后）实现手机端查看
- CDP 穿透 Shadow DOM 点击发布按钮

```bash
cd draft-xhs
./scripts/xhs-browser-publish --title "标题" --content "正文" --images-dir ./照片/
```

详细文档见各 skill 目录下的 `SKILL.md`。