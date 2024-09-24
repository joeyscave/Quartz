---
publish: false
date created: 2024-01-09
date modified: 2024-04-01
---
+ 付款方式
	+ 支付宝周期扣款
		+ 需[注册公司](注册公司.md)，且有对公账号
		+ [产品介绍 - 支付宝文档中心 (alipay.com)](https://opendocs.alipay.com/open/20190319114403226822/intro)
	+ 文档中放二维码收款，凭支付截图
	+ 稳定后推出季付
	+ 国内提供订阅服务的平台
		+ 小报童 - 需要 3k 粉丝，不足需审核
		+ 知识星球 - 网页版不佳
		+ 面包多
			+ 无法提供周期扣款
			+ 有一系列营销组件
			+ 抽成 13%
			+ 用户付费需跳转面包多登录
	+ Example
		+ 机场
			+ 应该是注册了个体工商户，接通了支付宝当面付 API，付款时收款人显示为 xxx 小卖部
		+ Spotify 合租：[声田合租-流媒体会员合租平台 (hezu.life)](https://hezu.life/?cid=1&mid=18)
			+ 这是怎么做到的，付款页面显示声田合租
			+ [淳安声田贸易商行 - 企查查 (qcc.com)](https://www.qcc.com/firm/32fb4820ba66668c8f30c600f1aa6ad2.html)
		+ Podwise
			+ lemonsqueezy - 可以支付宝或微信付款
	+ Resources
		+ [小朴在做独立开发(twitter.com)](https://twitter.com/leohuntercn/status/1771476605489303604)
		+ [请问大家，独立开发者怎么在海外收款？ · w2solo - 独立开发者社区](https://w2solo.com/topics/2941)
		+ 

+ 文案
	+ 非合租，不是共享账号
		+ 不会泄露使用信息
		+ 没有炸号风险
	+ 人性化的退款约定
		+ 如果因故障无法使用，立即退当月月费，且修复后当月免费
	+ **需要科学上网** （部署在 Vercel 上）
+ 部署流程
	+ 新建 Vervel Project
	+ ~~选择自己库里 fork 的 test 库~~
	+ 增加环境变量
	+ ![[z-source/Pasted image 20240109173749.png]]
+ copilot 限额：800 RPM per Dataverse environment [(Quotas)](https://learn.microsoft.com/en-us/microsoft-copilot-studio/requirements-quotas)
+ 其他发展可能性
	+ 在已有客户端（如 OpenCat）上加 copilot 接口
	+ 在除 web 的其他服务上包装，例如 chatgpt-on-wechat
+ 用户画像
	+ 开箱即用，无需获取 Copilot API 或者进行部署，尽量降低用户心智成本
+ 市场调查：GPT 使用方式

+ 用户量
	+ 100 人 - 3k MRR - 自给自足
	+ 1000 人 - 3w MRR - 大厂薪资
	+ 10000 人 - 30w MRR - 财富自由
+ 这个项目或许也可以用 GPT4 PLUS 做 worker，与 copilot 混合使用
+ 机场使用的系统框架：[Anankke/SSPanel-Uim](https://github.com/Anankke/SSPanel-Uim?tab=readme-ov-file)
+ 友商
	+ [头顶冒火 (burn.hair)](https://burn.hair/)
+ 问题
	+ ![](CleanShot%202024-04-01%20at%2013.05.png)

### 架构

+ 项目架构：[gpt4 架构 | onemodel](https://www.onemodel.app/d/H3YWoNapZh7BJl3UYLXRa)

### copilot-gpt4-service （CGS）

+ copilot-gpt4-service （CGS）：[aaamoon/copilot-gpt4-service](https://github.com/aaamoon/copilot-gpt4-service)
	+ 不建议方式
		+ 以公共服务的方式提供接口（让用户自己填入 copilot API）
			+ 多个 Token 在同一个 IP 地址进行请求，容易被判定为异常行为
		+ 同客户端 Web(例如 ChatGPT-Next-Web) 以默认 API 以及 API Key 的方式提供公共服务
			+ 同一个 Token 请求频率过高，容易被判定为异常行为
		+ Serverless 类型的提供商进行部署
			+ 服务生命周期短，更换 IP 地址频繁，容易被判定为异常行为
	+ ==需要解决的问题==
		+ 认证方面，防止一号多用
		+ ==搞清楚 copilot 具体限额，主要两方面==
			+ 一个 ip 能发送几个 token
			+ 一个 token 的发送频率限制
		+ 获取多个通过学生认证的 github 账号
		+ Web 端使用时，回答和设置中的模型一致，这是为什么？
			+ 因为 CGS 原样转发请求中传入的model
			+ 如果在 web 中取消加入系统级消息，会始终回答是 gpt3
				+ 即便是 OpenAI API 也会出现这种情况
		+ 设置两道 rate limiter
			+ 一道在用户 → Web （此时 header 带访问密码）
			+ 一道在 Web → CGS （此时 header 带 copilot key）
		+ 其实完全可以不用 Vercel，把 copilot key 写死在 CGS，用 super token 做用户鉴权，这样`[custom_url, supertoken]`就是对外的接口，可以实现对 web 的分离
		+ ==关键是确定一个 copilot key 究竟能给几个人用==
		+ 每个 copilot api key 能生效多久？
			+ 我的第一个 copilot key 是 [2024-01-18](2024-01-18.md) 生成的，说明至少两个月有效
	+ 每个服务器上部署有限个 CGS（每个 CGS 使用不同的 copilot API）
	+ 对每个 token 设置一个总 rate limiter
	+ 把 copilot API 写在 CGS 中，web 中可以随便填一个
	+ 研究一下`服务配置`一节
	+ 模型支持情况可能有更新（最近开始支持 GPT4-turbo）
	+ 即使不能使用 Copilot 也可以获取 token，这相当于是一个 device token

### chatgpt-next-web （Web）

+ chatgpt-next-web：[ChatGPTNextWeb/ChatGPT-Next-Web](https://github.com/ChatGPTNextWeb/ChatGPT-Next-Web)
	+ 使用 node 做后端，为 api 路由提供响应
	+ 所有数据保存在用户浏览器本地
	+ 访问密码可以有多个，由此实现单个 Vercel 部署对多用户


### 销售

+ 找用户
	+ GPT4 拼车

### Archive

+ 解决用 NextChat 在 Vercel 部署不泄露 API
	+ 按照这个 issue 中的方案在 fork 的 test 库中更改代码：
	+ [如何支持非OPENAI官方的API和KEY · Issue #3427](https://github.com/ChatGPTNextWeb/ChatGPT-Next-Web/issues/3427)
	+ 利用 BASE_URL 环境变量把自己的 api 和地址写到后端
	+ 看 nginx 日志发现即便 BASE_URL 写了端口，还是会访问 :443
		+ 更改 nginx 的 conf.d 文件，调换 memos 和 api 的端口
	+ 这个方法目前仅能在设置里不显示 API, 但是实际上请求直接发送给阿里云服务器了，域名和 API 在 network 里一目了然
	+ ==3.13 重新拿官方库部署了一下，发现 BASE_URL 的问题已经解决了==