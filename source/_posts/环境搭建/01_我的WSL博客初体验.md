---
title: 01_技术博客网站搭建
date: 2025-09-13 17:26:18
tags:
categories: 环境搭建
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

## 5. hexo使用
```
hexo new "我的WSL博客初体验" # 创建新文章
hexo generate # 生成静态文件
hexo server # 启动本地服务器
hexo deploy # 部署到远程
hexo clean # 清除缓存和已生成文件​​ (遇到奇怪问题时先用这个)
hexo list <type> # 列出文章、标签等
```

## 6. cloudbase部署网站
### 6.1 安装cli
https://console.cloud.tencent.com/
```
npm install -g @cloudbase/cli --registry=https://registry.npmmirror.com
tcb
```
### 6.2 CI/CD流水线
`vim .github/workflows/deploy.yml`
```
name: Deploy Hexo

on:
  push:
    branches:
      - main

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout source
        uses: actions/checkout@v4
        with:
          submodules: true

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Cache dependencies
        uses: actions/cache@v4
        with:
          path: node_modules
          key: ${{ runner.os }}-node-${{ hashFiles('package-lock.json') }}
          restore-keys: |
            ${{ runner.os }}-node-

      - name: Install dependencies
        run: npm ci

      - name: Generate
        run: npx hexo generate

      - name: Deploy to Tencent Cloud
        uses: TencentCloudBase/cloudbase-action@v2
        with:
          secretId: ${{ secrets.SECRET_ID }}   
          secretKey: ${{ secrets.SECRET_KEY }} 
          envId: ${{ secrets.ENV_ID }}         

```
```git push```
- 仓库>>setting>>secrets>>添加 SECRET_ID SECRET_KEY ENV_ID
- SECRET_ID SECRET_KEY通过用户>>账号信息>>访问密钥>>API密钥管理获取
- 查看secrets日志调试流程