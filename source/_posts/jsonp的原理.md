---
title: jsonp的原理
---

- `jsonp : json with padding`

- 利用的主要是`script`标签的`src`属性天生就有跨域的能力
		
- `jsonp`不是一个新的技术 更多的是前端和后端的约定

- 前端要做的事情

	+ 1.在全局环境下准备一个全局函数 用来接收数据

	+ 2.在接口地址中通过`callback`参数将函数的名字传递给后端

	+ `<script src="http://www.example.com?callback=全局函数的名字"></script>`

- 后端要做的事情

	+ 1.通过`$_GET['callback']`取到接口中传递过来的函数名字

	+ 2.通过一系列操作将数据获取到

	+ 3.给前端返回 函数的调用 并且将数据通过函数参数的形式传递进函数     