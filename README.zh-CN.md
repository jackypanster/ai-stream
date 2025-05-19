# 使用 GitHub Pages 部署 Hugo 博客

本文档详细介绍了如何使用 GitHub Pages 设置和部署 Hugo 博客。它还包括如何编写和发布新博客文章的说明。

## 前提条件

* 一个 GitHub 账户
* 本地机器上安装了 Git
* 本地机器上安装了 Hugo (推荐使用 extended 版本)

## 设置

1.  **创建一个新的 Hugo 站点：**

    ```bash
    hugo new site ai-stream
    cd ai-stream
    git init
    git submodule add [https://github.com/adityatelange/hugo-PaperMod.git](https://github.com/adityatelange/hugo-PaperMod.git) themes/PaperMod
    ```

    \* 将 `ai-stream` 替换为您所需的站点名称。
    \* 此示例使用 "PaperMod" 主题。

2.  **初始化或更新子模块：**

    ```bash
    git submodule update --init --recursive
    ```

    \* 这个命令至关重要，以确保主题（和任何其他子模块）被正确初始化或更新。

3.  **配置站点：**

    * 编辑根目录中的 `config.toml` 文件以配置您的站点设置（标题、baseURL 等）。
    * **重要：** 正确设置 `baseURL`。
        * 对于用户/组织站点：`baseURL = "https://<您的用户名>.github.io/"`
        * 对于项目站点：`baseURL = "https://<您的用户名>.github.io/<您的仓库名>/"`

4.  **选择并应用主题：**

    * 如果您没有使用 starter site，请将主题添加到站点的 `themes` 目录中。
    * 修改 `config.toml` 以指定主题：`theme = "<主题名称>"` （在本例中，`theme = "PaperMod"`）

5.  **创建您的第一篇文章：**

    ```bash
    hugo new content posts/my-first-post.md
    ```

    \* 这将在 `content/posts/` 目录中创建一个新的 Markdown 文件用于您的文章。

## 部署到 GitHub Pages

1.  **创建一个 GitHub 仓库：**

    * 在 GitHub 上创建一个新的仓库来存储您的 Hugo 站点的源代码（例如，`ai-stream`）。

2.  **将您的本地代码推送到 GitHub：**

    ```bash
    git add .
    git commit -m "Initial commit"
    git branch -M main
    git remote add origin [https://github.com/](https://github.com/)<您的用户名>/<您的仓库名>.git
    git push -u origin main
    ```

    \* 将 `<您的用户名>` 和 `<您的仓库名>` 替换为您的 GitHub 信息。

3.  **配置 GitHub Pages 部署（使用 GitHub Actions）：**

    * 在 `.github/workflows/hugo.yaml` 创建一个新的 workflow 文件：

    ```yaml
    name: Hugo Build & Deploy to GitHub Pages

    on:
      push:
        branches:
          - main

    jobs:
      build-deploy:
        runs-on: ubuntu-latest
        steps:
          - uses: actions/checkout@v3
            with:
              submodules: true
              fetch-depth: 0

          - name: Setup Hugo
            uses: peaceiris/actions-hugo@v2
            with:
              hugo-version: 'latest'
              extended: true

          - name: Build
            run: hugo --minify

          - name: Deploy to GitHub Pages
            uses: peaceiris/actions-gh-pages@v3
            with:
              github_token: ${{ secrets.GITHUB_TOKEN }}
              publish_dir: ./public
              cname: # (可选) 您的自定义域名，例如，'example.com'
    ```

    * 这个 workflow 将会：
        * 检出您的代码。
        * 设置 Hugo。
        * 构建您的站点。
        * 将 `public` 目录部署到 `gh-pages` 分支。

4.  **启用 GitHub Pages：**

    * 转到您的仓库的 Settings 选项卡。
    * 找到 "Pages" 部分。
    * 在 "Source" 下，选择 "Deploy from a branch" 并选择 `gh-pages` 分支。
    * 保存设置。

5.  **触发部署：**

    * 将任何更改推送到您的 `main` 分支。这将触发 GitHub Actions workflow 来构建和部署您的站点。

6.  **访问您的站点：**

    * Workflow 完成后，您的站点将可通过以下 URL 访问：
        * 对于用户/组织站点：`https://<您的用户名>.github.io/`
        * 对于项目站点：`https://<您的用户名>.github.io/<您的仓库名>/`

## 编写新的博客文章

1.  **创建一个新的文章文件：**

    ```bash
    hugo new content posts/my-new-post.md
    ```

    \* 这将创建 `content/posts/my-new-post.md`。

2.  **编辑文章文件：**

    * 在您的编辑器中打开 Markdown 文件。
    * 在前置元数据（文件开头的 YAML 格式）中添加元数据：

    ```yaml
    ---
    title: "我的新文章"
    date: 2025-05-20 # 文章的日期
    draft: false     # 设置为 true 以隐藏文章
    tags: ["标签1", "标签2"]
    categories: ["分类1"]
    ---

    这是我的新博客文章的内容。
    您可以在此处使用 Markdown 语法。
    ```

3.  **编写您的文章内容：**

    * 使用 Markdown 格式化您的文章。

4.  **保存文件并推送到 GitHub：**

    ```bash
    git add content/posts/my-new-post.md
    git commit -m "添加我的新文章"
    git push origin main
    ```

    * 这将触发 GitHub Actions workflow 重新构建并将您的站点与新文章一起重新部署。

## 更新您的站点

* 要更新站点的内容，只需编辑 `content/` 目录中的 Markdown 文件，然后将更改推送到您的 `main` 分支。
* 要更新站点的设计或主题，请修改主题文件或站点的配置，然后推送更改。

## 使用的主题

* 此设置使用 [PaperMod](https://github.com/adityatelange/hugo-PaperMod) 主题。

此设置提供了一个自动化的 workflow，用于使用 GitHub Pages 发布和更新您的 Hugo 博客。