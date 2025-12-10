---
title: How to Build Personal Blog by Quartz 4
draft: "false"
tags:
  - blog
  - "#quartz"
created: 2025-12-09 23:40
modified: 2025-12-10 21:05
---
Quartz 4 is so brilliant to build personal blog easily.
You can follow me to set the most useful and simplest blog, or go with [Welcome to Quartz 4](https://quartz.jzhao.xyz/) for more details.

## 0. Before Start
You need
1. Github account (Free!)
2. A computer
3. Network(Maybe VPN is necessary)
4. [Obsisian]([Obsidian - Sharpen your thinking](https://obsidian.md/))(Optional, but I think it's the best note editor)

Have a look to my blog: [Welcome to Sisyphus Mountain](https://sisyphus-mountain.github.io/)

## 1. Set Environment
## git
https://git-scm.com/install/windows

## Node.js
[Node.js — Download Node.js®](https://nodejs.org/zh-cn/download)
Get a prebuilt Node.js for Windows is enough.

## Validate Installation
Open PowerShell or CMD:
```bash
git --version
node --version
npm --version
```

## 2. Set Quartz

Select a parent folder you like,  and click right to open Terminal.
Then,
```bash
git clone https://github.com/jackyzha0/quartz.git
cd quartz
npm i
npx quartz create
```

Select `Empty Quartz`, `Shortest path(default)`.

Now, we are at `/quartz`, enter the sub folder `/content`, and you can see `/content/index.md`, which is the home page.

It's very recommended that you use Obsidian and set the folder `/content` as your vault.

## 3. Write Your First Blog

```md
---
title: Example Title
draft: false
tags:
  - example-tag
---

The rest of your content lives here. You can use **Markdown** here :)

```
## 4. Build and Preview

Open `/quartz/quartz.config.ts`,
edit `pageTitle: "Quartz 4"` to `pageTitle: "<Your Blog Name>"`

Run the command at `/quartz` to build Blog locally.

```bash
npx quartz build --serve
```

Open a web browser and visit `http://localhost:8080/` to view it.

## 5. Set Github Repository

Create a new repository on GitHub.com. Do **not** initialize the new repository with `README`, license, or `gitignore` files.

For a shorter domain, name the repository as `<username>.github.io`, which will be your Blog domain name.

Copy the remote repository URL, like `https://github.com/<username>/<username>.github.io.git`, which is `REMOTE_URL`.

Run several command at `/quartz`

```bash
git remote -v
git remote set-url origin <REMOTE-URL>
git remote add upstream https://github.com/jackyzha0/quartz.git
```

Upload your repository,

```bash
npx quartz sync --no-pull
```

## 6. Hosting

In your local Quartz, create a new file `quartz/.github/workflows/deploy.yml`.

```yml
name: Deploy Quartz site to GitHub Pages

on:
  push:
    branches:
      - v4

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: false

jobs:
  build:
    runs-on: ubuntu-22.04
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0 # Fetch all history for git info
      - uses: actions/setup-node@v4
        with:
          node-version: 22
      - name: Install Dependencies
        run: npm ci
      - name: Build Quartz
        run: npx quartz build
      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: public

  deploy:
    needs: build
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

Then,

1. Head to “**Settings**” tab of your forked repository and in the sidebar, click “**Pages**”. Under “**Source**”, select “**GitHub Actions**”.
2. Run command  at `/quartz` to commit:

```bash
npx quartz sync
```

This should deploy your site to `<username>.github.io`.

## 7. Comments

### Install giscus
[GitHub Apps - giscus](https://github.com/apps/giscus)

Install for selected repositories `<username.github.io`.

Head to “**Settings**” tab of your forked repository and Scroll down to the "**Features**" section and select **Discussions**.

Open [giscus](https://giscus.app/zh-CN) and Input `<username>/<username>`
Make sure you select `Announcements` for the Discussion category.

Then, you will see your arguments of repo discussion,

```html
<script src="https://giscus.app/client.js"
        data-repo="john-ferrel/john-ferrel.github.io"
        data-repo-id="R_kgDOQmNBbA"
        data-category="Announcements"
        data-category-id="DIC_kwDOQmNBbM4Czn4C"
        data-mapping="pathname"
        data-strict="0"
        data-reactions-enabled="1"
        data-emit-metadata="0"
        data-input-position="bottom"
        data-theme="preferred_color_scheme"
        data-lang="en"
        crossorigin="anonymous"
        async>
</script>
```


Open `quartz/quartz.layout.ts`, and edit
```ts
afterBody: [
  Component.Comments({
    provider: 'giscus',
    options: {
      // from data-repo
      repo: 'john-ferrel/john-ferrel.github.io',
      // from data-repo-id
      repoId: 'R_kgDOQmNBbA',
      // from data-category
      category: 'Announcements',
      // from data-category-id
      categoryId: 'DIC_kwDOQmNBbM4Czn4C',
      
      lang: 'en'
    }
  }),
],
```

