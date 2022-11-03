---
title: "Hugo之Wercker停服解决方案-Github Actions"
date: 2022-11-02T20:45:43+08:00
lastmod: 2022-11-02T20:45:43+08:00
categories: ["hugo"]
tags: ["hugo"]
keywords: ["Wercker", "hugo", "CI", "Github Actions"]
toc: false
---

昨晚发现使用Wercker更新博客时一直build失败，而且没有错误提示（撒币Oracle）！！！

折腾了一晚上，今早准备好好看看Wercker文档的时候发现：

> **Important Notice:** Wercker service will be shutdown on **October 31st, 2022** . We recommend you to use [OCI DevOps service](https://docs.oracle.com/en-us/iaas/Content/devops/using/getting_started.htm). Thank you for using Wercker.
>
> **Note:** [OCI DevOps](https://www.oracle.com/devops/devops-service/) doesn't have a free build resource for free-tier "always free" account types; if you migrate to OCI DevOps, you will need a paid Oracle Cloud account. You can upgrade to a paid Oracle Cloud account by adding a payment method. OCI offers "Pay As You Go" if your usage is small. For larger accounts, please contact one of our Oracle [Sales](https://cloud.oracle.com/account-management/payment-method) team members for assistance.

Wercker停服了！！！

不让白嫖了，还想让人迁移到收费的，想得美。果断放弃Wercker，找免费CI服务的时候，发现了Github Actions。

如果跟我一样只是用来构建部署hugo发布博客，那么我**墙裂推荐Github Actions**。

**这种情况下，Wercker到Github Actions非常简单，甚至完全不需要了解Github Actions。**

![我裂开了](/image/Hugo之Wercker停服解决方案-GithubActions/1.png)

只需要修改为使用Github Actions，然后按照指引生成一个yml文件就完成了。甚至生成的yml不需要任何修改。

这是我生成的：

```yaml
# Sample workflow for building and deploying a Hugo site to GitHub Pages
name: Deploy Hugo site to Pages

on:
  # Runs on pushes targeting the default branch
  push:
    branches: ["main"]

  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:

# Sets permissions of the GITHUB_TOKEN to allow deployment to GitHub Pages
permissions:
  contents: read
  pages: write
  id-token: write

# Allow one concurrent deployment
concurrency:
  group: "pages"
  cancel-in-progress: true

# Default to bash
defaults:
  run:
    shell: bash

jobs:
  # Build job
  build:
    runs-on: ubuntu-latest
    env:
      HUGO_VERSION: 0.102.3
    steps:
      - name: Install Hugo CLI
        run: |
          wget -O ${{ runner.temp }}/hugo.deb https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_Linux-64bit.deb \
          && sudo dpkg -i ${{ runner.temp }}/hugo.deb
      - name: Checkout
        uses: actions/checkout@v3
        with:
          submodules: recursive
      - name: Setup Pages
        id: pages
        uses: actions/configure-pages@v2
      - name: Build with Hugo
        env:
          # For maximum backward compatibility with Hugo modules
          HUGO_ENVIRONMENT: production
          HUGO_ENV: production
        run: |
          hugo \
            --minify \
            --baseURL "${{ steps.pages.outputs.base_url }}/"
      - name: Upload artifact
        uses: actions/upload-pages-artifact@v1
        with:
          path: ./public

  # Deployment job
  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v1

```

改完之后，hugo构建的代码不在gh-pages分支了，如果想看构建后的代码，可以在Actions选项下具体的workflow里Artifacts压缩包看。
