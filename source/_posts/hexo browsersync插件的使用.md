---
title: hexo browsersync插件的使用
---

### browsersync 的作用

> Browsersync能让浏览器实时、快速响应您的文件更改（html、js、css、sass、less等）并自动刷新页面。更重要的是Browsersync可以同时在PC、平板、手机等设备下进项调试。

### 如何在 Hexo 中使用 browsersync  

- 第一步
 + 打开 cmd 或 git 切换到你的 blog 主目录下
- 第二步
 + 执行 `$ npm install hexo-browsersync --save`
- 第三步
 + 执行 `$ hexo server`
- 第四步
 + 访问 `http://localhost:4000/` ,开始编写你的代码或文档

执行完上述步骤,你可以一边编写源文档,一边查看编译后的文档

当然,当你编写完,关闭 `http://localhost:4000/` 之后,你还需要执行 `$ hexo deploy --generate`,刷新你的博客页面,这样你的文章就上传了

如有需要,请点击 [github 的 hexo brwsersync](https://github.com/hexojs/hexo-browsersync)
