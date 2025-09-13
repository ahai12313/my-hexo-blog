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
      - main # 或者您的博客源代码所在的分支名

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout source
      uses: actions/checkout@v2
      with:
        submodules: true

    - name: Setup Node.js
      uses: actions/setup-node@v2
      with:
        node-version: 14 # 或者您选择的 Node.js 版本

    - name: Cache dependencies
      uses: actions/cache@v2
      with:
        path: ~/.npm
        key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
        restore-keys: |
          ${{ runner.os }}-node-

    - name: Install dependencies
      run: npm ci

    - name: Generate
      run: npx hexo generate

    - name: Deploy to Tencent Cloud
      uses: TencentCloudBase/github-action-cloudbase-deploy@v1
      with:
        secret_id: ${{ secrets.SECRET_ID }}
        secret_key: ${{ secrets.SECRET_KEY }}
        env_id: ${{ secrets.ENV_ID }}

```
`git push`
仓库>>setting>>secrets>>添加 SECRET_ID SECRET_KEY ENV_ID
SECRET_ID SECRET_KEY通过用户>>账号信息>>访问密钥>>API密钥管理获取