注意：撰写中……

# Class fields 提案的问题报告

- 提交者：@hax
- 360TC状态：尚未讨论

本报告总结了 class fields 提案的情况和存在的问题，将提交给 360 Web前端TC 审阅和讨论。

作者：贺师俊（hax）

## 关于本报告中立性的说明

作者在本报告中会尽力秉持客观中立的态度，但需要提请审阅者注意：

- 本人从2018年年初开始就深度参与了该提案的公开讨论，并在讨论中逐渐从中立转向反对该提案的立场。本人寻求 360 作为 TC39 成员对该提案动用否决权。
- 本人与TC39中的部分代表对许多问题存在严重分歧，这些分歧不仅包括具体的技术issue，也包括更深入的层面，如动机、需求、优先级、心智模型、设计准则、流程合理性等。

有鉴于此，审阅者不应预设本报告的所有内容都是完全中立无偏的，并应对本报告涉及问题的争议性程度有所理解。

为了避免作者有意无意的非中立对审阅者造成影响，经作者请求，由 赵文博 担任“独立信源”。赵文博 自6月12日到7月18日期间，预先自行阅读原始材料（包括 class fields 提案和相关提案的 repo 中的内容、TC39会议纪要中的相关内容等），并不可接触任何由作者主导提供的材料（包括并不限于以 class fields 提案为主题的内部和外部演讲、文章等，但作者在 TC39 repo 的 issue 中参与讨论所发内容除外），以避免因受作者影响而产生先入为主偏见的潜在风险。

当审阅者对 class fields 提案的任何具体问题存在疑问，经过充分交流，本人的解释和赵文博的解释仍然存在矛盾的，请以赵文博的解释为准。

## Class fields 简介

### Public fields

以下直接引用 [class fields 提案](https://github.com/tc39/proposal-class-fields)的 README 中的代码示例。

定义一个Counter组件，每按一次显示数字+1，按ES2015方式书写：
```js
class Counter extends HTMLElement {
  clicked() {
    this.x++;
    window.requestAnimationFrame(this.render.bind(this));
  }

  constructor() {
    super();
    this.onclick = this.clicked.bind(this);
    this.x = 0;
  }

  connectedCallback() { this.render(); }

  render() {
    this.textContent = this.x.toString();
  }
}
window.customElements.define('num-counter', Counter);
```

按照 class fields 提案的 public fields 特性（早期称为 class instance properties），可写成：
```js
class Counter extends HTMLElement {
  x = 0;

  clicked() {
    this.x++;
    window.requestAnimationFrame(this.render.bind(this));
  }

  constructor() {
    super();
    this.onclick = this.clicked.bind(this);
  }

  connectedCallback() { this.render(); }

  render() {
    this.textContent = this.x.toString();
  }
}
window.customElements.define('num-counter', Counter);
```

这个例子的变动只有一行，即原本在构造器中的最后一行代码`this.x = 0;`被移动到了class头部，并改为带有初始化的 field 声明`x = 0;`。另一种可能性是，构造器的代码不变，在class头部加上不带有初始化的`x;`声明。

使用 field 声明的好处是，增加了代码的自说明性；同时，当初始化代码可以一并被转移到声明中时，构造器中可以减少状态变化的代码。

仅仅从这个例子来说，可能优点也并不明显。public field [早期](https://github.com/tc39/proposal-class-public-fields/tree/ce435b2be94f2816f0d6729024e413dfbe89b0f4)是一份单独提案，我们可以追溯当时的示例。

一个简单的React组件：
```js
class ReactCounter extends React.Component {
  constructor(props) { // boilerplate
    super(props); // boilerplate

    // Setup initial "state" property
    this.state = {
      count: 0
    };
  }
}
```
可以注意到需要一些对于React开发者来说纯粹是 boilerplate 的代码。

使用 public fields 之后：
```js
class ReactCounter extends React.Component {
  state = {
    count: 0
  };
}
```
对于React组件来说，绝大多数情况下可以不再需要显式的书写构造器。

以下是一个更完整的例子：

```js
class ReactCounter extends React.Component {
	constructor(props) {
		super(props)

		this.state = {count: 0}

		this.handleClick = this.handleClick.bind(this)
	}

	handleClick() {
		this.setState(state => ({count: state.count + 1}))
	}

	render() {
		return (
			<button onClick={this.handleClick}>{this.state.count}</button>
		)
	}
}
```

使用 public fields 之后：
```js
class ReactCounter extends React.Component {
	state = {count: 0}

	handleClick = () => {
		this.setState(state => ({count: state.count + 1}))
	}

	render() {
		return (
			<button onClick={this.handleClick}>{this.state.count}</button>
		)
	}
}
```

可以注意到，不仅不再需要显式的书写构造器，通过使用箭头函数，也避免了手动bind。

简言之，public fields 能显著改善React的class组件的书写和阅读体验。当然，对于一般的类继承来说，也有简化的效果，[一些介绍文章也强调了这一点](https://v8.dev/features/class-fields#simpler-subclassing)。

### Private fields

以下仍引用 class fields 提案的 README 中的代码示例。

这是我们之前已经看到的使用 public fields 的示例：
```js
class Counter extends HTMLElement {
  x = 0;

  clicked() {
    this.x++;
    window.requestAnimationFrame(this.render.bind(this));
  }

  constructor() {
    super();
    this.onclick = this.clicked.bind(this);
  }

  connectedCallback() { this.render(); }

  render() {
    this.textContent = this.x.toString();
  }
}
window.customElements.define('num-counter', Counter);
```

在上述代码中，`x` 是外部可访问的（比如可以通过 `document.querySelector('num-counter').x`访问），如果希望将这一实现细节隐藏，可以使用 private fields 特性：
```js
class Counter extends HTMLElement {
  #x = 0;

  clicked() {
    this.#x++;
    window.requestAnimationFrame(this.render.bind(this));
  }

  constructor() {
    super();
    this.onclick = this.clicked.bind(this);
  }

  connectedCallback() { this.render(); }

  render() {
    this.textContent = this.#x.toString();
  }
}
window.customElements.define('num-counter', Counter);
```

可以看到，public fields 和 private fields 的区别在于 field 的名称包含了 `#` 前缀。`#x`对于class外部代码来说是不可见的。（Decorator 提案会提供能力允许在`#x = 0`声明之前加上特定的 decorator 声明，让该decorator的代码可以访问`#x`的值。由于 decorator 声明必须是在 class 内的，意味着只有 class 的作者可以决定向什么代码暴露`#x`。）

Private fields 提供了强封装，阻止 class 的用户依赖内部实现细节。

### 语义

为方便理解，我使用代码转换来解释 class fields 的语义。

```js
class A {
	a
}
class B extends A {
	b = 1
}
```
大体等价于
```js
class A {
	constructor() {
		Object.defineProperty(this, 'a', {
			value: undefined,
			configurable: true,
			enumerable: true,
			writable: true,
		})
	}
}
class B extends A {
	constructor(...args) {
		super(...args)
		Object.defineProperty(this, 'b', {
			value: 1,
			configurable: true,
			enumerable: true,
			writable: true,
		})
	}
}
```

而
```js
class Counter {
	#x = 0
	inc() {
		++this.#x
	}
	value() {
		return this.#x
	}
}
```
大体等价于
```js
const $x_store = new WeakMap()
function $init_x(instance, value) {
	$x_store.set(instance, value)
}
function $read_x(instance) {
	if (!$x_store.has(instance)) throw new TypeError()
	return $x_store.get(instance)
}
function $write_x(instance, value) {
	if (!$x_store.has(instance)) throw new TypeError()
	$x_store.set(instance, value)
}

class Counter {
	constructor() {
		$init_x(this, 0)
	}
	inc() {
		$write_x(this, $read_x(this) + 1)
	}
	value() {
		return $read_x(this)
	}
}
```

简言之，public fields 本质上是 own properties，private fields 本质上是 WeakMap。

### 相关提案

[Private methods and getter/setters](https://github.com/tc39/proposal-private-methods) 和 [Static class features](https://github.com/tc39/proposal-static-class-features) 是独立的提案，但是都基于 class fields 提案的语法和语义。代码示例如下：

Private methods 和 getter/setters：
```js
class Counter extends HTMLElement {
  #xValue = 0;

  get #x() { return #xValue; }
  set #x(value) {
    this.#xValue = value;
    window.requestAnimationFrame(this.#render.bind(this));
  }

  #clicked() {
    this.#x++;
  }

  constructor() {
    super();
    this.onclick = this.#clicked.bind(this);
  }

  connectedCallback() { this.#render(); }

  #render() {
    this.textContent = this.#x.toString();
  }
}
window.customElements.define('num-counter', Counter);
```


## Class fields 的显著问题

以下讨论 class fields 提案存在的显著问题

### public fields 的语法二义性

```js
function test() {
  [x] = [1, 2, 3] // 解构
}

class Test {
  [x] = [1, 2, 3] // 计算属性
}
```

```js
function test() {
  name = 'hax'
  greeting = `Hello ${name}!`
}

class Test {
  name = 'hax'
  greeting = `Hello ${name}!`
}
```

### public fields 的 ASI 问题

### Define own property 语义与基于原型的继承机制的冲突（[[Define]] vs [[Set]]）

###

...

## 相关链接

- https://github.com/tc39/proposal-class-fields
- https://github.com/tc39/proposal-private-methods
- https://github.com/tc39/proposal-static-class-features/
- https://github.com/tc39/proposal-class-public-fields/tree/c3d2f4da26dde486018aaffcd4bf91615cedfa1f
- https://github.com/tc39/proposal-class-public-fields/tree/ce435b2be94f2816f0d6729024e413dfbe89b0f4
- https://v8.dev/features/class-fields
