# DreamZhongJu 的个人主页

这是一个无需构建的静态个人主页与博客起始模板，已包含首页、作品集、文章页和“网站正在建设中”的提示。

## 发布到 GitHub Pages

1. 在 GitHub 新建一个公开仓库，名称必须是 `DreamZhongJu.github.io`。
2. 将本目录的全部文件上传到仓库根目录（包括隐藏的 `.github` 文件夹）。
3. 在仓库的 **Settings → Pages** 中，选择 **GitHub Actions** 作为发布来源。
4. 推送后等待工作流完成，网站将发布到 `https://dreamzhongju.github.io`。

## 更新内容

- 修改 `index.html`：主页介绍、项目和最新文章卡片。
- 修改 `blog/index.html`：文章列表。
- 修改 `styles.css`：视觉样式。

后续可将每一篇文章增加为 `blog/文章-slug/index.html`，并从文章列表链接过去。
