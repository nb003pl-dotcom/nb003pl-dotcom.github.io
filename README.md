# AI × 金融研究笔记

用 Hugo、PaperMod 和 Giscus 构建的 GitHub Pages 站点。

## 首次发布

1. 在 GitHub 新建一个公开仓库，不要初始化 README。
2. 在本目录执行：

   ```bash
   git add .
   git commit -m "feat: initialize Hugo PaperMod site"
   git branch -M main
   git remote add origin git@github.com:<用户名>/<仓库名>.git
   git push -u origin main
   ```

3. 打开仓库的 **Settings → Pages**，在 **Build and deployment → Source** 选择 **GitHub Actions**。
4. 等待 `Deploy Hugo site to Pages` 工作流完成。工作流会自动采用 GitHub Pages 给出的正确站点 URL，因此用户站点和项目站点都适用。

## 配置 Giscus

1. 确认仓库为公开仓库，并在 **Settings → General → Features** 开启 Discussions。
2. 安装 [Giscus GitHub App](https://github.com/apps/giscus) 并授权此仓库。
3. 打开 [giscus.app/zh-CN](https://giscus.app/zh-CN)，输入仓库名并选择 `Announcements`（公告）分类。
4. 把页面生成的 `data-repo`、`data-repo-id`、`data-category`、`data-category-id` 分别填入 `hugo.yaml` 的：

   ```yaml
   params:
     giscus:
       repo: "用户名/仓库名"
       repoId: "R_..."
       category: "Announcements"
       categoryId: "DIC_..."
   ```

字段为空时评论区不会加载，文章页面的其他内容不受影响。

## 本地预览

需要 Hugo Extended 0.146.0 或更高版本；GitHub Actions 当前固定使用 0.169.0：

```bash
git submodule update --init --recursive
hugo server -D
```

然后访问 <http://localhost:1313/>。

## 发布新文章

```bash
hugo new content posts/my-new-post.md
```

编辑文章并把 front matter 中的 `draft` 改为 `false`，提交并推送到 `main` 后会自动部署。
