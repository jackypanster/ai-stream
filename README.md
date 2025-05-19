# Setup and Deployment of a Hugo Blog with GitHub Pages

This document provides a detailed guide on setting up and deploying a Hugo blog using GitHub Pages. It also includes instructions on how to write and publish new blog posts.

## Prerequisites

* A GitHub account
* Git installed on your local machine
* Hugo installed on your local machine (extended version is recommended)

## Setup

1.  **Create a new Hugo site:**

    ```bash
    hugo new site ai-stream
    cd ai-stream
    git init
    git submodule add [https://github.com/wowchemy/starter-hugo-academic.git](https://github.com/wowchemy/starter-hugo-academic.git) themes/starter-hugo-academic
    ```

    \* Replace `ai-stream` with your desired site name.
    \* You can replace the theme with any other Hugo theme. This example uses "starter-hugo-academic".

2.  **Configure the site:**

    * Edit the `config.toml` file in the root directory to configure your site settings (title, baseURL, etc.).
    * **Important:** Set the `baseURL` correctly.
        * For a user/organization site: `baseURL = "https://<your-username>.github.io/"`
        * For a project site: `baseURL = "https://<your-username>.github.io/<your-repository-name>/"`

3.  **Choose and apply a theme:**

    * If you didn't use a starter site, add a theme to your site's `themes` directory.
    * Modify `config.toml` to specify the theme: `theme = "<theme-name>"`

4.  **Create your first post:**

    ```bash
    hugo new content posts/my-first-post.md
    ```

    \* This creates a new Markdown file for your post in the `content/posts/` directory.

## Deployment to GitHub Pages

1.  **Create a GitHub repository:**

    * Create a new repository on GitHub to store your Hugo site's source code (e.g., `ai-stream`).

2.  **Push your local code to GitHub:**

    ```bash
    git add .
    git commit -m "Initial commit"
    git branch -M main
    git remote add origin [https://github.com/](https://github.com/)<your-username>/<your-repository-name>.git
    git push -u origin main
    ```

    \* Replace `<your-username>` and `<your-repository-name>` with your GitHub information.

3.  **Configure GitHub Pages deployment (using GitHub Actions):**

    * Create a new workflow file at `.github/workflows/hugo.yaml`:

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
              cname: # (Optional) Your custom domain, e.g., 'example.com'
    ```

    * This workflow will:
        * Checkout your code.
        * Set up Hugo.
        * Build your site.
        * Deploy the `public` directory to the `gh-pages` branch.

4.  **Enable GitHub Pages:**

    * Go to your repository's Settings tab.
    * Find the "Pages" section.
    * Under "Source," select "Deploy from a branch" and choose the `gh-pages` branch.
    * Save the settings.

5.  **Trigger the deployment:**

    * Push any change to your `main` branch. This will trigger the GitHub Actions workflow to build and deploy your site.

6.  **Access your site:**

    * After the workflow completes, your site will be available at:
        * For user/organization sites: `https://<your-username>.github.io/`
        * For project sites: `https://<your-username>.github.io/<your-repository-name>/`

## Writing New Blog Posts

1.  **Create a new post file:**

    ```bash
    hugo new content posts/my-new-post.md
    ```

    \* This creates `content/posts/my-new-post.md`.

2.  **Edit the post file:**

    * Open the Markdown file in your editor.
    * Add metadata in the front matter (YAML format at the beginning of the file):

    ```yaml
    ---
    title: "My New Post"
    date: 2025-05-20 # Date of the post
    draft: false     # Set to true to hide the post
    tags: ["tag1", "tag2"]
    categories: ["category1"]
    ---

    This is the content of my new blog post.
    You can use Markdown syntax here.
    ```

3.  **Write your post content:**

    * Use Markdown to format your post.

4.  **Save the file and push to GitHub:**

    ```bash
    git add content/posts/my-new-post.md
    git commit -m "Add my new post"
    git push origin main
    ```

    * This will trigger the GitHub Actions workflow to rebuild and redeploy your site with the new post.

## Updating Your Site

* To update your site's content, simply edit the Markdown files in the `content/` directory and push the changes to your `main` branch.
* To update your site's design or theme, modify the theme files or your site's configuration and push the changes.

This setup provides an automated workflow for publishing and updating your Hugo blog with GitHub Pages.