# function implementation hiding

提案状态

* Stage: 2
* Repo: https://github.com/tc39/proposal-function-implementation-hiding
* Champion: @michaelficarra

本文是对`function-implementation-hiding`提案目前现状的总结，基本翻译自提案的 README（译者：李冬杰 @makeco）。该提案的champion正在寻求推进该提案到stage 3（2019年12月的会议上暂时没有达到stage 3，明年2月TC39会议可能重新审议）。一旦成为stage 3的提案，修改成本会非常高。

本提案对运行时安全、运行时内省和元编程、运行时错误收集等方面有比较重大的影响，希望这篇介绍能引起更多人思考，大家有关心的问题尽早提出来，让JS变得更好。

## 简介

函数实现隐藏，这个提案在目前已有的指令`use strict`基础上，增加两个新指令`hide source`和`sensitive`（这两个名字是暂时的，在这个[issue #3](https://github.com/tc39/proposal-function-implementation-hiding/issues/3)里有讨论，目前大家比较能接受这两个名字），它提供了一种方式，让开发者可以控制某些实现细节不应暴露给用户，举个例子：

```javascript
function foo() { /* ... */ }

console.assert(foo.toString().contains('...'))
```

如果我们使用了`hide source`指令

```javascript
’hide source‘

function foo() { /* ... */ }

console.assert(!foo.toString().contains('...'))
```

添加这个指令有什么好处呢？对于某些开源项目的作者来说，他们不希望用户在使用时会依赖具体实现的细节，否则在重构后可能会发生breaking change。还有一些对安全比较敏感的项目，通过隐藏代码实现细节可以规避一些问题，还有polyfill的作者希望实现“高保真”的polyfill，他们可能出于不同的目的都想将代码的具体实现细节对使用者隐藏。

`hide source`通过隐藏`Function.prototype.toString`的输出，并且隐藏`Error.prototype.stack`中的文件属性和位置信息达到隐藏实现细节的目的。`sensitive`同样会隐藏`Function.prototype.toString`的输出，并且完全从`Error.prototype.stack`中省略了函数。随着新的安全问题出现，`sensitive`指令的功能可能会被进一步扩展。


## 解决什么问题？

### Function.prototype.toString

JavaScript的`Function.prototype.toString`保留着实现该函数的源代码，这就让调用者得到不必要的能力，能够观察到函数的实现细节，他们可以内省函数的实现并对它作出反应，这就会导致项目作者一些无害的重构，却给使用者造成了breaking change的麻烦。使用者甚至可以从源码中提取一些比较隐秘的值，这严重破坏了应用程序的封装。举个例子，Angular依赖`f.toString()`输出的源码内省出函数参数名，然后作为框架依赖注入功能的一部分，这就会导致刚才提到的问题。

另一个问题是通过`f.toString()`的输出可以判断函数是否为native实现，比如`console.log(Math.random.toString())`输出中的字符串中包含`[native code]`而不是函数的源码，而自定义的函数就会输出源码，这样就会让”高保真“polyfill变得困难，为此有些polyfill的开发者会[替换](https://github.com/zloirock/core-js/blob/9f051803760c02b306aae2595621bb7ef698fc29/modules/_redefine.js#L28)`Function.prototype.toString`或者[实现自己的](https://github.com/paulmillr/es6-shim/blob/8d7aec1403751686dbbd3c4fa13a7bb584a75bf3/es6-shim.js#L139)`polyfillFn.toString()`。


### Error.prototype.stack

JavaScript的（事实上是非标准的）`Error.prototype.stack` getter 展示了存在或不存在堆栈信息情况下的调用行为。对于递归调用函数，会在堆栈展示递归调用的次数。如果调用行为依赖了某些秘密的值，无论是在源码层面还是词法层面都会导致这个秘密的值被部分或者全部暴露，另外它还会带出一些文件属性的位置信息，和 `toString`以类似的方式影响着重构。


## 解决方法

解决方正文开始就谈到了，使用`hide source`或者`sensitive`指令，改变函数`toString()`的输出从而阻止其暴露出具体的实现细节，这两个指令像`use strict`一样可以用在整个文件或者某个函数中，同样是向下作用，从而使其**作用域内的所有内容**以及在函数作用域内的**函数本身**都被隐藏实现细节，举个例子：

```javascript
function foo() {
  const x = () => {};

  const y = () => {
    "hide source";
    class Z {
      m() {}
    }
  };
}
```

在这个例子中`foo`和`x`都不会被隐藏实现细节，而`y`和`Z`以及`Z.prototype.m`都会被隐藏实现细节。

## 为什么选择使用指令实现？

该提议很大程度上借鉴了[JavaScript现有的指令支持的优势](https://tc39.github.io/ecma262/#directive-prologue)：

1. 可以简单的从文件级隐藏实现细节，通过在文件开头加`hide source`或者`sensitive`。
2. 可以用匿名函数包裹，从而将非隐藏实现细节和隐藏实现细节的代码绑定在一起。
3. 向后兼容，可以轻松地部署代码，从而尽力达到隐藏实现细节的目的，在未实现此提案的引擎中，指令不生效。
4. 方便工具简单、静态的决定一个函数是否应该被隐藏实现细节。

## 那些被拒绝掉的方案

### A one-time hiding function

这个方案引入了`Error.hideFromStackTraces` 和 `Function.prototype.hideSource`两个函数，通过调用函数从而实现对目标函数实现细节的隐藏。

```javascript
function foo() { /* ... */ }

console.assert(foo.toString().includes("..."));

foo.hideSource();

console.assert(foo.toString() === "function foo() { [ native code ] }");
```

这种方式比指令方案要差一些：

- 很难做到一次性隐藏所有函数的实现细节，指令可以做到文件级隐藏
- 可以隐藏所有人的函数，并不只是你自己创建和控制的函数
- 非词法的，一些作用在源代码上的工具要依赖启发式的技术来决定函数的实现是否应该被隐藏
- 这个属性作用在函数上而不是整个源代码上，抽象级别有错误，并且很难推理。

### A clone-creating hiding function

和上面的方法类似，不过`foo.hideSource()`返回的是一个被隐藏实现细节的函数，和上面的方法有相同的缺点，还有其他原因：

- 克隆函数难以说明和解释，可以看看关于`toMethod()`方法的讨论，这是一个从ES2015开始的提案，该方法进行了一次函数克隆，但由于其复杂性而被TC39拒绝了。
- 一些函数很难被克隆函数完全替代，这些方法的执行需要依赖他们正确的上下文环境。

### delete Function.prototype.toString

有人提出了这种方案，在源文件中先执行`delete Function.prototype.toString`，但是这不适用于任何多领域环境，看下面的例子：

```javascript
delete Function.prototype.toString;

function foo() { /* ... */ }

const otherGlobal = frames[0];
console.assert(otherGlobal.Function.prototype.toString.call(foo).includes("..."));
```

而且这是一种非常钝的方式，只能在领域级别内使用，可能只有应用程序的开发人员使用。而指令的作用目标是库文件的开发者，应用级别的开发者可以使用out-of-bound的解决方案(后面会提到)。

而且这个提案中的`sensitive`指令将来可能会扩展，不仅是`delete Function.prototype.toString`隐藏源码这么简单的事情，考虑到该方案扩展性较差，也被否定了。

### 使用Symbol作为开关

这种方法看起来很诱人，就像`Symbol.isConcatSpreadable`或者`Symbol.iterator`，但是它和第一种方案有相同的问题，而且还是可逆的，完全可以关闭。或者你想说我们可以把它设计成true的时候隐藏实现细节，设置为false什么都不做，我觉得这样会被diss。

```javascript
function foo() { /* ... */ }
console.assert(foo.toString.includes("..."));

foo[Symbol.hideSource] = true;
console.assert(!foo.toString.includes("..."));

foo[Symbol.hideSource] = false;
console.assert(foo.toString.includes("...")); // oops
```

## 常见问题

### 该提案是否应该隐藏函数名和函数参数个数？

不会，因为JavaScript已经有隐藏函数名和函数参数个数的机制，而且这个功能并不是`hide source`指令的目标需求，所以这个提案不会包含该行为。

```javascript
function foo(a, b, c) { }
console.assert(foo.name === "foo");
console.assert(foo.length === 3);

// 通过defineProperty隐藏name和参数数量
Object.defineProperty(foo, "name", { value: "" });
Object.defineProperty(foo, "length", { value: 0 });
console.assert(foo.name === "");
console.assert(foo.length === 0);
```

### 对devtools和其他审查函数实现的方法有什么影响？

这个提案是针对JS的提案，只会影响JS代码的行为，devtools不考虑在内，这个提案也没有想过要改变devtools的行为。因此通过提案的指令被隐藏实现的函数，可以被任何具有特权的API审查到，比如devtools使用的API。

目前该提案只关心两件事：

1. Function.prototype.toString() 输出的源码字符串
2. Error.prototype.stack 输出的错误栈信息

下面这些是提案不考虑的：

1. 使用`console.log(function () {})`打印出来的内容
2. 使用`console.log(new Error)`打印出来的内容
3. unhandleed exception 或 unhandleed rejection输出的结果
4. 使用devtools看到源码或者HTTP响应的内容
5. 在devtools上断点调试或者在函数中因uncaught exception暂停的信息
6. 其他...

这些都是和浏览器实现有关的，devtools团队只需要考虑用户体验，不用管JS的规范。

### 会不会造成开发者在所有地方都使用它呢？

比较乐观的态度认为是不会的，`hide soruce` 不像 `use strict` 会让代码更好。`hide source` 对那些想要更高封装和自由度重构代码的作者来说，是一种特殊的机制。

有开发者觉得这两个指令可以节省内存，因为`Function.prototype.toString`隐藏源码后，这部分代码可以直接删掉从而不占内存。`sensitive`作用下`Error.prototype.stack`可以省略隐藏了实现细节的函数，这样又会节省一部分内存，如果真的会节省内存，我想大部开发者都会选择使用指令吧，因为使用指令没什么坏处，还能优化性能，但实际上并没有这么乐观，在最后面我们会讨论这个问题。


### 为什么没有`preserve source`指令？

我们好像确实需要一个在使用`hide source`的函数内部，通过使用`preserve source`保留函数实现的细节，但是这种场景并不多，你可以把函数提取出来从而避免使用这个指令。但是对于直接调用`eval`和反射类型的情况，我们没法对文本做任何（可以提取出来的）假设，也就无法提取函数声明了。


## 外部设置的全局节省内存开关

TC39的历史中有很多想法、动机、提案关于节省内存，在2018年1月份的会议上，TC39的委员们意识到有两个关于这方面的提案：

1. 一种封装机制，与源代码一起in-bound使用，特别适合于库。（这个提案）
2. 一种节省内存机制，out-of-bound源代码使用，特别适合应用。

第二种提案的背后考虑是因为，引擎为了让`Function.prototype.toString`返回预期的结果，会占用大量内存保存源码和其所有的依赖项，如果存在一种方式让应用层开发者全局关闭源码存储，这样会节省大量内存。这种方式取决于应用运行的环境，如果是浏览器可以用meta标签，Node.js环境可以用flag。

2018 TC39的会议后，决定将这个想法分成两部分落实，其中一个就是这个提案，另一个是根据宿主环境的out-of-bound方式，但是不久之后和引擎实现者讨论后发现，这套节省内存的机制前提是存在缺陷，对于那些保留源码进行懒编译的引擎来说，用户的一个隐藏源码的指令并不会节省内存。

因此out-of-bound节省内存的开关到现在都没推进，如果浏览器引擎实现者改变了他们懒加载编译的技术，该提案附录将会更新并指导有兴趣做champion的同学进行这方面的工作。
