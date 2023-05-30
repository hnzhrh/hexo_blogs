---
title: Obsidian + GitHub + Hexo 笔记博客工作流搭建
date: 2023-05-30 00:41
tags:
- tools
- IT
categories:
- tools
---
#tools

# 安装 Obsidian

下载 `AppImage` 文件，` chmod ` 加可执行权限后安装，在 `~/.bashrc ` 中添加命令别名：
```shell
alias obsidian=~/Downloads/Obsidian-1.2.8.AppImage
```

插件推荐：

```text
better-fn
cm-editor-syntax-highlight-obsidian
cmenu-plugin
dataview
easy-typing-obsidian
highlightr-plugin
link-favicon
number-headings-obsidian
obsidian-advanced-slides
obsidian-auto-link-title
obsidian-excalidraw-plugin
obsidian-footnotes
obsidian-image-auto-upload-plugin
obsidian-kanban
obsidian-minimal-settings
obsidian-plantuml
obsidian-style-settings
obsidian-timeline
quickadd
remember-cursor-position
table-editor-obsidian
templater-obsidian
```

笔记模板：

```text
---
title: need to changed
date:  <% tp.file.creation_date() %>
tags:
-  <% tp.file.folder() %>
- IT
categories:
-  <% tp.file.folder() %>
---
# <% tp.file.folder() %>
```

# 安装 Hexo

```shell
npm install hexo-cli -g
```

`Hexo` 工作流：

```shell
mkdir ~/hexo_blog
cd ~/hexo_blog

hexo init #初始化博客目录
hexo npm install

# cp Obsidian 笔记到Hexo博客中
cp ./测试内容.md ~/hexo_blog/source/_posts/

hexo g #生成静态文件
hexo server #本地调试，端口4000
```

# 安装 `hexo-theme-aurora` 主题
根据官网进行配置，删除一些不需要的组件，有个坑就是评论模块，需要改下代码，配置静态资源压缩，尽量减少传输文件的大小。

# 同步到 `Github`  从仓库

`GitHub` 建立仓库，把本地 `Hexo` 根目录整个推送上去，注意删掉 `.gitignore` 文件的内容。

配置 `GitHub Action` 间参考资料，需要注意一个点，如果是上传了依赖包的，需要看下自己本地 `node` 版本，与 `Action` 脚本 `Node` 版本保持一致。

# 发布脚本
写了个脚本用来把 `Obsidian` 笔记复制到 `Hexo` 博客，并推送到 `GitHub`， `GitHub` 触发 `Action` 推送到阿里云 `OSS` 。

```shell
#!/bin/sh

file=$1
new_file_name=$2
hexo_posts_path=/home/erpang/hexo_blog/source/_posts

echo "copy $file to Hexo workspace..."
cp "$file" "$hexo_posts_path/$new_file_name"

cd /home/erpang/hexo_blog
echo "Git add..."
git add .

echo "Git commit..."
git commit -m "Posts update."

echo "Git push..."
git push

```

修改 `~/.bashrc` 添加别名：`alias deploy_blogs='sh ~/deploy_blogs.sh'`

Demo：`deploy_blogs ./Obsidian\ +\ GitHub\ +\ Hexo\ 笔记博客工作流搭建.md Obsidian_GitHub_Hexo_notes_blogs_workflow.md`

# 参考资料
* [Github Actions自动部署hexo博客到阿里云OSS - 掘金](https://juejin.cn/post/6987568619739676708)
* [Hexo](https://hexo.io/zh-cn/)
* [Obsidian](https://obsidian.md/)
* [GitHub - auroral-ui/hexo-theme-aurora: 🏳️‍🌈 Futuristic auroral Hexo theme.](https://github.com/auroral-ui/hexo-theme-aurora)
* [hexo Aurora 删除评论侧边栏 | 生生's Blog](https://gongxuanzhang.github.io/post/aurora%E5%88%A0%E9%99%A4%E8%AF%84%E8%AE%BA%E4%BE%A7%E8%BE%B9%E6%A0%8F)
* [Hexo 博客静态资源压缩优化 | 虾丸派](https://www.playpi.org/2018112101.html)

