---
title: 搭建Hexo博客和配置主题的一些坑
date: 2016-09-06 23:15:37
tags:
---
今天更新了Hexo版本，并更换了博客主题，在配置这些环境的过程中遇到了一些坑，在此分享一波!
## 无法更新或生成Hexo
* 问题:node版本升级得过高，导致与现有运行库不兼容
* 解决方案:用npm安装n,用n来回顾node版本号

```sh
sudo npm i -g npm

n *.*.*    (切回到*.*.*稳定版本)

node -v
```
<!-- more -->
## 添加Next主题
```sh
mkdir themes/next

curl -s https://api.github.com/repos/iissnan/hexo-theme-next/releases/latest | grep tarball_url | cut -d '"' -f 4 | wget -i - -O- | tar -zx -C themes/next --strip-components=1

```
## 按照文档分别Hexo和Next中的_config.yml
注意:后面有一个空格。下面是我的一些配置项
* Hexo
```sh
sidebar: 
 offset: 16

deploy:
  type: git
  repo: ssh://git@github.com/chengsluo/chengsluo.github.io
  branch: master
```
* Next
```sh
scheme: Gemini

mathjax:
  enable: true
```

## 常用命令
```sh
### 编写文档
hexo new <title>
## 重新生成
hexo clean
hexo generate
## 调试预览
hexo server
## 调试完成后再发布
hexo deploy
## 一键发布
hexo clean && hexo generate --deploy

## 添加远程git部署支持
npm install hexo-deployer-git --save
## 启用本地搜索
npm install hexo-generator-search --save
## 摘要与正文分割
<!-- more -->
```
## 参考资料

hexo官方文档
https://hexo.io/zh-tw/index.html

Hexo博客使用MathJax公式并解决Markdown渲染冲突问题
https://zhuanlan.zhihu.com/p/33857596

NexT主题
https://github.com/iissnan/hexo-theme-next

Node version management
https://github.com/tj/n