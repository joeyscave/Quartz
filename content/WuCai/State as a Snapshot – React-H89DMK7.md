---
标题: "State as a Snapshot – React"
笔记ID: H89DMK7
笔记类型: page
星标: false
tags: 
域名: react.dev
域名2: react.dev
作者: ""
原文链接: "https://react.dev/learn/state-as-a-snapshot"
五彩链接: "https://marker.dotalk.cn/#/?noteidx=H89DMK7"
划线数量: 0
创建时间: 2024-03-24 14:07
更新时间: 2024-03-24 14:24
---

## Notes
1. 重新渲染时使用提交渲染时 state 变量的快照
1. 重新渲染一定会等到 event 处理函数执行完再执行
2. 重新渲染即调用当前组件函数
3. 执行到 setState 会提交渲染排队，然后继续执行下去，所以 setState 后的函数会先于重新渲染
4. 在每次渲染中 state 变量的值永不变，setState 改变 state 变量的值只会作用于下个渲染

## Highlights
