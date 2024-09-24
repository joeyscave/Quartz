---
标题: "Queueing a Series of State Updates – React"
笔记ID: H89DM6M
笔记类型: page
星标: false
tags: 
域名: react.dev
域名2: react.dev
作者: ""
原文链接: "https://react.dev/learn/queueing-a-series-of-state-updates"
五彩链接: "https://marker.dotalk.cn/#/?noteidx=H89DM6M"
划线数量: 0
创建时间: 2024-03-24 14:29
更新时间: 2024-03-24 14:29
---

## Notes
1. 等到 event 处理函数执行结束，react 的队列里攒了好多 setState，这时就可以批处理这些变量更改，只触发一次重新渲染

## Highlights
