---
created: 2025-12-24T03:55:07+08:00
modified: 2025-12-24T19:28:23+08:00
feature: thumbnails/external/70dfd27858ca3c39732765125e7130fc.png
thumbnail: thumbnails/resized/c65b86c47d386b5a9f29e79138b3cff4_b89e22fb.jpg
tags:
  - blog
title: 搭建Hugo个人博客
date: 2025-12-23T19:55:06.587Z
lastmod: 2025-12-24T12:40:02.382Z
---
# 本地windows部署

下载hugo，解压，一定要下载extend后缀的版本\
命令：

```sh
hugo new site 网站名
```

反馈：

```bash
Just a few more steps...

1. Change the current directory to D:\apply\pro\Hugo\hugo_0.153.2_windows-amd64\WzhUEDev.
2. Create or install a theme:
   - Create a new theme with the command "hugo new theme <THEMENAME>"
   - Or, install a theme from https://themes.gohugo.io/
3. Edit hugo.toml, setting the "theme" property to the theme name.
4. Create new content with the command "hugo new content <SECTIONNAME>\<FILENAME>.<FORMAT>".
5. Start the embedded web server with the command "hugo server --buildDrafts".
```

到hugo主题仓库找一个主题：[hugo themes](https://themes.gohugo.io/)

> \[!example]- 好看的主题\
> [博丽灵梦](https://themes.gohugo.io/themes/hugo-theme-reimu/)\
> [个人主页](https://themes.gohugo.io/themes/blox-tailwind/)\
> [作品主页](https://themes.gohugo.io/themes/demius/)\
> [博客主页](https://themes.gohugo.io/themes/hextra/)\
> [播客主页-小红书](https://themes.gohugo.io/themes/hugo-theme-dream/)\
> [可能有很强自定义](https://themes.gohugo.io/themes/hugo-theme-next/)\
> [系列文章](https://themes.gohugo.io/themes/e25dx/)\
> [有分类](https://pehtheme-hugo.netlify.app/)

## 直接使用模板

找到模板链接，如：`https://github.com/D-Sketon/hugo-reimu-template`\
使用模板创建仓库\
pull新仓库\
模板的主题只有文件夹，是空仓库，把主题克隆在该位置

## 常规方案

复制hugo.exe到创建的仓库文件夹\
将主题解压复制到themes文件夹，去掉版本号\
![](https://cdn.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/20251224044110230.png)

修改配置文件 hugo.toml,使用新主题\
此时最好对照主题的样例项目修改

```C++
theme = 'hugo-theme-reimu'
```

找到主题的示例(\_example)代码，复制到content目录下

启动项目

```sh
hugo server
```

打开网站即可

## 各种问题

让其支持原生html\
关闭安全模式：hugo.toml

```text
[markup.goldmark.renderer]
      unsafe = true
```

## Ob发布插件

[hugo publish](https://github.com/kirito41dd/obsidian-hugo-publish)\
![](https://cdn.jsdelivr.net/gh/cnwenzhihong/ImageHosting/ProjectMarkdown/20251224161920407.png)
