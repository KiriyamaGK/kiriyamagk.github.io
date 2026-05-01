---
title: Hexo-GitHub-博客搭建教程
date: 2026-05-01
tags: [笔记, 教程]
categories: 工具
---

# Hexo + GitHub Pages 博客搭建记录

## 一、环境准备

### 1. 安装 Node.js

从官网下载安装 LTS 版本：https://nodejs.org/

安装完成后重启终端，验证：

```bash
node -v
npm -v
```

### 2. 安装 Hexo

```bash
npm install -g hexo-cli
```

### 3. 初始化博客

```bash
hexo init my-blog
cd my-blog
npm install
```

### 4. 本地预览

```bash
hexo server
```

访问 http://localhost:4000 即可预览。

## 二、写作与发布

### 1. 创建新文章

```bash
hexo new "文章标题"
```

会在 source/_posts/ 下生成一个 .md 文件。

### 2. Front-matter

每篇文章开头需要包含以下格式：

```markdown
---
title: 文章标题
date: 2026-05-01
tags: [标签1, 标签2]
categories: 分类名称
---
```

### 3. 部署到 GitHub Pages

安装部署插件：

```bash
npm install hexo-deployer-git --save
```

修改根目录的 _config.yml：

```yaml
deploy:
  type: git
  repo: https://github.com/你的用户名/你的用户名.github.io.git
  branch: main
```

执行部署（PowerShell 中用分号代替 &&）：

```bash
hexo clean; hexo generate; hexo deploy
```

访问 https://你的用户名.github.io 即可看到博客。

## 三、多端编辑同步

使用 Git 分支同步：

main 分支：存放生成的静态网页

hexo 分支：存放源文件

在电脑 A 上初始化

```bash
git checkout -b hexo
```

创建 .gitignore：

```bash
echo "node_modules/
public/
.deploy*/
db.json" > .gitignore
```

提交并推送：

```bash
git add .
git commit -m "首次备份源文件"
git push origin hexo
```

在电脑 B 上克隆

```bash
git clone -b hexo https://github.com/你的用户名/你的用户名.github.io.git my-blog
cd my-blog
npm install
```

日常更新流程

```bash
# 写作前拉取最新源文件
git pull origin hexo

# 写文章... 然后部署
hexo clean; hexo generate; hexo deploy

# 备份源文件
git add .
git commit -m "添加新文章"
git push origin hexo
```

## 四、注意事项

好像不能设置为私有仓库，否则博客网站会404（流汗黄豆）
