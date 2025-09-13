---
title: 技术博客网站搭建
date: 2025-09-13 17:26:18
tags:
---

- 开发平台：vscode + wsl(ubuntu)

## 1. 更新软件包
```
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl git
```
## 2. 安装node.js
```
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
```

```
# 验证安装
node -v
npm -v
```

## 3. Hexo
```
# 安装Hexo CLI
sudo npm install -g hexo-cli
```

```
# 初始化博客
hexo init ~/my-blog  # 初始化到用户目录下的 my-blog 文件夹
cd ~/my-blog
npm install           # 安装依赖
```

```
# 启动本地预览 hexo server
http://localhost:4000/
```
## 4. 安装推荐扩展
- Markdown All in One​​（Markdown 增强）
- Hexo Utils​​（Hexo 辅助工具，可选）

## 5. 新建文章
`hexo new "我的WSL博客初体验"`