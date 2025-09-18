---
title: 03_hexo常用方法
date: 2025-09-18 21:42:37
tags:
categories: 环境搭建
---
## 常用方法
```
hexo new "我的WSL博客初体验" # 创建新文章
hexo generate # 生成静态文件
hexo server # 启动本地服务器
hexo deploy # 部署到远程
hexo clean # 清除缓存和已生成文件​​ (遇到奇怪问题时先用这个)
hexo list <type> # 列出文章、标签等
```

## 给文章分类
1. 给文章添加分类`categories: 环境搭建`
2. 获取主题`git clone https://github.com/hexojs/hexo-theme-landscape themes/landscape`
3. themes/landscape/_config.yml配置文件中添加
```
menu:
  Home: /
  Archives: /archives
  Categories: /categories
```
4. 刷新缓存`hexo clean && hexo g && hexo s`


## 启动服务器
`hexo clean && hexo g && hexo s`