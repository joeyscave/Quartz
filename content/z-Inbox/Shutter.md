---
publish: false
date created: 2024-01-06
date modified: 2024-03-07
created: 2024-01-06
modified: 2024-05-11
---
+ 简单插入了两个 iframe，结果拒绝连接
	+ 为什么：同源策略
		+ [浏览器的同源策略 - Web 安全 | MDN (mozilla.org)](https://developer.mozilla.org/zh-CN/docs/Web/Security/Same-origin_policy#%E8%B7%A8%E6%BA%90%E7%BD%91%E7%BB%9C%E8%AE%BF%E9%97%AE)
		+ [X-Frame-Options - HTTP | MDN (mozilla.org)](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/X-Frame-Options)
	+ 解决
		+ 发现 google 在 url 中插入 igu=1 就可以解决，但似乎并不可持续
			+ [html - Original source of igu=1 parameter to display google in iframe - Stack Overflow](https://stackoverflow.com/questions/67306800/original-source-of-igu-1-parameter-to-display-google-in-iframe)
		+ google 可编程搜索引擎似乎是可持续方案
			+ [Introduction  |  Programmable Search Engine  |  Google for Developers](https://developers.google.com/custom-search/docs/tutorial/introduction)
		+ ==Allow X-Frame-Options 插件==
			+ 作用是移除 http 响应头中的 X-Frame-Options
		+ 必应只要科学上网且 ip 地址合适就可以连接
+ 禁止滑动事件向浏览器传播，确保 iframe 内的左右滑动不会前进后退
	+ 效果不明确，暂时弃用
+ 自动化部署
	+ github webhooks
	+ 服务器在 8083 端口后台启动一个 js 脚本监听处理 webhook
	+ gitee webhooks
	+ 使用 `nohup node ./Shutter.js &` 运行监听脚本，其中 `nohup`的作用是分离进程和会话，使得退出 ssh 之后后台运行的脚本不退出
+ 改为托管在 Gitee
+ Kagi 拒绝连接
	+ 在改写 Allow X-Frame-Options 插件, 使得 Content-Security-Policy 也被移除之后，Kagi 显示登录页面
	+ 目前认为是 cookie 中的 samesite=Lax 阻止了 cookie 复用导致的

+ 窗口管理的思路可借鉴：[Home | Noi (nofwl.com)](https://noi.nofwl.com/)
	+ 没开源，挂在 github 的是 landingpage

+ 出现了一个竞品：[煎蛋搜索](https://www.iamaming.com/)
	+ 体验做的很奇怪，只有电商可以在同一页显示

