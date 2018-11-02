---
title: express-start
---
### express

基于 `Node.js` 平台，快速、开放、极简的 Web 开放框架

####安装

假定你已经安装了 node.js，接下来为你的应用创建一个目录，然后进入此目录并将其作为当前工作目录

```bash
$ mkdir myapp
$ cd myapp
```

通过 `npm init` 命令为你的应用创建一个 `package.json` 文件

```bash
$ npm init
```

一路 enter ,最后 yes

接下来在 `myapp` 目录下安装 Express 并将其保存到依赖列表中

```bash
$ npm install express --save
```

#### hello world

在 index.js 文件里：

```bash
const express = require('express')
const app = express()

app.get('/',(req,res) => {
    res.send('HELLO WORLD')
})

app.listen(3000,() => {
    console.log('Example app listening on port 3000!')
})
```

运行一下

```bash
$ node index.js
```

访问 `http://localhost:3600`，就可以看到输出

#### express-generator

可以快速创建一个应用的骨架

首先需要全局安装

```bash
$ npm install express-generator -g
```

例如，如下命令创建了一个名称为 *myapp* 的 Express 应用。此应用将在当前目录下的 *myapp* 目录中创建，并且设置为使用 [Pug](https://pugjs.org/) 模板引擎（view engine

```bash
$ express --view=pug myapp
```

```bash
$ cd myapp
$ npm i
$ npm start
```

一个框架，完成搭建服务

​	