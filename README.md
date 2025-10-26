# hexo使用说明

## 编写文档
hexo new <title>
## 重新生成
hexo clean
hexo generate

## 完善主题配置，包括about等
cp node_modules/hexo-theme-next/_config.yml _config.next.yml

## 调试预览
hexo server
## 调试完成后再发布
hexo deploy

## 一键发布
hexo clean && hexo deploy

## 按照deploy-git
npm install hexo-deployer-git --save

## 启用本地搜索
npm install hexo-generator-search --save