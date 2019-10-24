# hashbang 提案 360 反馈后跟踪

- 提案阶段：stage-3
- 反馈 issue：[#18](https://github.com/tc39/proposal-hashbang/issues/18)
- 国内反馈：[#2](https://github.com/75team/tc39/issues/2)

## 回顾

[hashbang 提案](https://github.com/tc39/proposal-hashbang) 允许脚本的第一行是如下形式：

```js
#!/usr/bin/env node
'use strict'
console.log(1)
```

即如果第一行是以`#!`开头的话，忽略第一行。这基本上是当前 node.js 等 CLI 的行为。

Hashbang 提案在 2018 年 3 月达到 stage 2，同年 11 月达到 stage 3，Chrome 74+ 和 Firefox 67+ 已实现。

经过 360TC 会议和国内社区反馈后，hax 发起[issue #18](https://github.com/tc39/proposal-hashbang/issues/18)，本文是反馈后 champion 的观点做摘要，方便国内社区讨论。

## 摘要

### 工具和服务可能导致的潜在风险

工具和服务对源码进行前置/追加/包裹/拼接注释或其他内容时，会导致该脚本解析失败。

**[Jordan Harband](https://github.com/ljharb)**：脚本的上下文环境(Browser、Node.js)假定了解析目标，任何工具处理后的代码都有颠覆源码作者意图的风险，并不只是添加 hashbang 才会发生脚本解析失败的情况，工具的开发者要对工具负责。注释和空格是代码的一部分，如果工具添加了这些导致了解析失败，那么一定是工具坏了，解决的方法应该是换更好的工具，而不是限制语言本身。

**[Bradley Farias](https://github.com/bmeck)** 对很多工具来说 hashbang 在所有位置都是合法的，不只是 CLI 脚本中，这证明了[这些风险](https://github.com/tc39/proposal-hashbang/issues/18#issuecomment-523193379)在社区已经不是问题了。在 hashbang 成为规范后，如果由于工具原因导致了解析错误，开发者可以选择不使用它或者解决工具存在的问题。

[详细内容](https://github.com/tc39/proposal-hashbang/issues/18#issuecomment-523071104)

### 可能的替代方案

- 将`#!`作为 ECMAScript 规范的一种注释
- 使用 SingleLineHTMLCloseComment`-->`替代 hashbang

**[Jordan Harband](https://github.com/ljharb)**：对委员会成员能够接受这样一种晦涩的注释表示怀疑；几乎没有人希望 HTML 注释继续存在了。

[详细内容](https://github.com/tc39/proposal-hashbang/issues/18#issuecomment-523181568)

### hashbang 放在任意位置

**[Bradley Farias](https://github.com/bmeck)**：如果要支持任意位置的 hashbang 应该创建新的 proposal，不应该在这个 issue 中讨论。

[详细内容](https://github.com/tc39/proposal-hashbang/issues/18#issuecomment-523193379)

### web 兼容性潜在问题

**[Bradley Farias](https://github.com/bmeck)**：如果证明存在 web 兼容性问题，应该有很多解决方法，但不应该在这个 issue 中讨论。

[详细内容](https://github.com/tc39/proposal-hashbang/issues/18#issuecomment-523193379)
