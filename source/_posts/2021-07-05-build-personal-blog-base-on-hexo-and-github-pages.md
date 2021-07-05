---
title: 基于 Hexo 和 Github Pages 服务搭建个人博客实践
date: 2021-07-05 20:39:05
tags: 
    - Hexo
    - Github Pages
categories:
    - 实践
---

## Hexo 和 Github Pages 是什么？

Hexo 是什么？

>Hexo 是一个快速、简洁且高效的博客框架。Hexo 使用 Markdown（或其他渲染引擎）解析文章，在几秒内，即可利用靓丽的主题生成静态网页。

Github Pages 是什么？

>GitHub Pages is designed to host your personal, organization, or project pages from a GitHub repository.

Pages 是设计来托管 Github 仓库里面页面的服务。

而我们要做的事情就是使用 Hexo 生成个人博客的页面，然后再利用 Github Pages 服务来托管。

## 使用 Hexo 生成个人博客页面

通过 npm 安装 ：

```sh
npm install -g hexo-cli
```

详细的安装教学可以看这里：[文档 | Hexo](https://hexo.io/zh-cn/docs/)

安装完毕后，初始化博客文件夹：

```sh
hexo init 3vilive-blog
cd 3vilive-blog
npm install
```

初始化完毕后，可以看到有这些文件和文件夹：

```
➜ ls
_config.landscape.yml node_modules          package.json          source                yarn.lock
_config.yml           package-lock.json     scaffolds             themes
```

我们主要关注这几项：

- `_config.yml` 是博客的配置
- `scaffolds` 是模板文件夹，新建文章的时候会使用里面的模板
- `source` 资源文件夹
- `themes` 主题文件夹

接着开始编辑博客配置（`_config.yml`），首先编辑博客的基本信息 (Site)：

```yaml
# Site
title: 3vilive's blog
subtitle: 'good luck have fun'
description: ''
keywords:
author: 3vilive
language: zh-CN
timezone: 'Asia/ShangHai'
```

接着设置 URL 这一栏：

```yaml
# URL
## Set your site url here. For example, if you use GitHub Page, set url as 'https://username.github.io/project'
url: https://3vilive.github.io
permalink: :year/:month/:day/:title/
permalink_defaults:
pretty_urls:
  trailing_index: false # Set to false to remove trailing 'index.html' from permalinks
  trailing_html: false # Set to false to remove trailing '.html' from permalinks
```

我们要使用 Github Pages，直接根据注释说明修改。

另外就是 `pretty_urls.trailing_index` 和 `pretty_urls.trailing_html` 我都设置为 `false`，因为不想在 URL 上看到 `html` 后缀。

为了管理方便，将 `new_post_name` 修改为为 `:year-:month-:day-:title.md
` 和 `post_asset_folder` 为 `true`：

```yaml
new_post_name: :year-:month-:day-:title.md
post_asset_folder: true 
```

这样每次创建创建新文章的时候，文件名字都会带上时间信息，并且会自动创建一个相对应的资源文件夹。

其他配置先不做修改，使用默认的值就行。

至此，博客的基本配置已经完成，我们可以运行命令在本地预览效果：

```
➜ hexo server
INFO  Validating config
INFO  Start processing
INFO  Hexo is running at http://localhost:4000 . Press Ctrl+C to stop.
```

打开 [http://localhost:4000](http://localhost:4000) 即可预览效果。


## 部署到 Github Pages 上

首先在 Github 上创建一个仓库，这里我创建了 `https://github.com/3vilive/3vilive.github.io`。

`hexo-deployer-git` 是 `Hexo` 的一个部署插件，我们安装 `hexo-deployer-git` 后就可以用 `Git` 来部署了:

```sh
npm install hexo-deployer-git --save
```

编辑配置文件 (`_config.yml`)，添加部署信息：

```
# Deployment
## Docs: https://hexo.io/docs/one-command-deployment
deploy:
  type: 'git'
  repo: https://github.com/3vilive/3vilive.github.io.git
  branch: gh-pages
```

执行部署命令：

```
hexo deploy
```

看到最后输出 `INFO  Deploy done: git` 的时候，说明部署完成了，这时候可以打开仓库主页，看看文件是否已经在仓库里面。 

点开 Settings -> Pages 页面，如果展示信息为 `Your site is published at https://3vilive.github.io/` 代表发布成功：


![](github-pages.png)


说明已经发布成功了，打开 [https://3vilive.github.io](https://3vilive.github.io) 验证一下。

![](blog-home.png)

## 开始写作和更新

使用 hexo new 命令创建新的文章，hexo new 的用法：

```
hexo new [layout] <title>
```

这里我创建了一篇文章《基于 Hexo 和 Github Pages 服务搭建个人博客实践》:

```
➜ hexo new post "build personal blog base on hexo and github pages"
INFO  Validating config
INFO  Created: ~/dev/3vilive-blog/source/_posts/2021-07-05-build-personal-blog-base-on-hexo-and-github-pages.md
```

打开对应的 `markdown` 文件就可以开始写作了，这里是 `~/dev/3vilive-blog/source/_posts/2021-07-05-build-personal-blog-base-on-hexo-and-github-pages.md`.

在写作期间，可以通过 `hexo server` 来预览最终效果。

写作完毕后，通过 `hexo deploy` 命令，就可以把文章发布啦。

## 自定义主题

默认的主题有点简陋，这里我找了一个主题还不错：

[https://github.com/fluid-dev/hexo-theme-fluid](https://github.com/fluid-dev/hexo-theme-fluid)


直接跟着里面的教程安装，最终效果如下：






## 参考

- [文档 | Hexo](https://hexo.io/zh-cn/docs/)