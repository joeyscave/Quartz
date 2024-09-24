---
标题: "Tutorial:Tic-Tac-Toe – React"
笔记ID: H8797A3
笔记类型: page
星标: false
tags: 
域名: react.dev
域名2: react.dev
作者: ""
原文链接: "https://react.dev/learn/tutorial-tic-tac-toe"
五彩链接: "https://marker.dotalk.cn/#/?noteidx=H8797A3"
划线数量: 0
创建时间: 2024-01-30 04:39
更新时间: 2024-01-30 04:44
date created: 2024-01-30
date modified: 2024-01-30
---

## Notes
+ 尽管 xIsNext 只被赋值一次，但是它的值会随着 currentMove 的改变而改变。这是因为在 React 中，每次组件的状态改变时，都会重新渲染组件，从而重新执行组件函数

+ 如果你在 Game 函数的主体中直接调用 setHistory（或者其他的状态设置函数，比如 setCurrentMove），而这个调用不是在事件处理器或者副作用函数（比如 useEffect 的回调函数）中，那么你的组件确实会陷入无限渲染循环。

## Highlights
