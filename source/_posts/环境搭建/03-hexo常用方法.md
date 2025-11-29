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

根据您提供的图片信息，Hexo 博客的文章排序方式如下：

## 如何自定义排序

### 1. 通过 Front-Matter 控制
在每篇文章的 Markdown 文件头部添加排序参数：
```yaml
---
title: 文章标题
date: 2025-11-18 14:00:00
updated: 2025-11-18 15:00:00
sticky: 100  # 置顶功能，数值越大越靠前
---
```

### 2. 修改主题配置文件
在主题的 `_config.yml` 中调整排序设置：
```yaml
# 按日期降序（默认）
index_generator:
  path: ''
  per_page: 10
  order_by: -date

# 按日期升序（最旧在前）
index_generator:
  path: ''
  per_page: 10
  order_by: date

# 按更新时间排序
index_generator:
  path: ''
  per_page: 10
  order_by: -updated
```

### 3. 常用排序选项
- `-date`：按创建时间降序（默认）
- `date`：按创建时间升序
- `-updated`：按更新时间降序
- `title`：按标题字母顺序
- `-title`：按标题字母倒序

### 4. 脚手架
scaffolds/post.md
hexo new effective-c "我的C++文章"
tcb hosting list -e cloud1-0g2dsbxh4872e179