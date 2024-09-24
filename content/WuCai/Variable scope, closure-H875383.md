---
标题: "Variable scope, closure"
笔记ID: H875383
笔记类型: page
星标: false
tags: 
域名: javascript.info
域名2: javascript.info
作者: ""
原文链接: "https://javascript.info/closure"
五彩链接: "https://marker.dotalk.cn/#/?noteidx=H875383"
划线数量: 5
创建时间: 2024-01-24 18:24
更新时间: 2024-01-24 18:43
---

## Notes


## Highlights
> When a Lexical Environment is created, a Function Declaration immediately becomes a ready-to-use function (unlike let, that is unusable till the declaration).
> That’s why we can use a function, declared as Function Declaration, even before the declaration itself.

> Now when the code inside counter() looks for count variable, it first searches its own Lexical Environment (empty, as there are no local variables there), then the Lexical Environment of the outer makeCounter() call, where it finds and changes it.
> #notes 相当于在 binding 一个函数的时候，把它的内部 Lexical Environment 也 binding 过去了，所以在外部调用的时候，查找某个变量，是从那个函数定义的地方的 scope 开始查找

> They analyze variable usage and if it’s obvious from the code that an outer variable is not used – it is removed.
> An important side effect in V8 (Chrome, Edge, Opera) is that such variable will become unavailable in debugging.

> Are counters independent?

> Is variable visible?

