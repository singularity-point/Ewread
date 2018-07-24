# 完美解释 Javascript 响应式编程原理
#article

> [深入响应式原理 — Vue.js](https://cn.vuejs.org/v2/guide/reactivity.html) <br>
> https://medium.com/vue-mastery/the-best-explanation-of-javascript-reactivity-fea6112dd80d <br>
> https://www.vuemastery.com/courses/advanced-components/evan-you-on-proxies/

很多前端 JavaScript 框架，包含但不限于（Angular，React，Vue）都拥有自己的响应式引擎。通过了解响应式变成原理以及具体的实现方式，可以提成对既有响应式框架的高效应用。

# 响应式系统
我们看一下 Vue 的响应式系统：

```html
<div id="app">
	<div>Price: ${{ price }}</div>
	<div>Total: ${{ price * quantity }}</div>
	<div>Taxes: ${{ totalPriceWithTax }}</div>
<div>
```

```js
<script src="https://cdn.jsdelivr.net/npm/vue"></script>
<script>
	var vm = new Vue({
	  el: '#app',
	  data: {
	    price: 5.00,
	    quantity: 2
	  },
	  computed: {
	    totalPriceWithTax() {
	      return this.price * this.quantity * 1.03
	    }
	  }
	})
</script>
```

Vue 知道每当 `price` 发生变换时，它做了如下三件事情：

* 在页面中更新 `price` 的值。
* 在页面中重新计算 `price * quantity` 表达式，并更新。
* 调用 `totalPriceWithTax` 方法，并更新。

但是等一下，这里有些疑惑，Vue 是怎么知道 `price` 更新了呢，如何去追踪更新的具体过程呢？

![](https://cdn-images-1.medium.com/max/1600/1*t8enMn6h0gjY6HNKoSVC1g.jpeg)

# 响应式在原生 JavaScript 中如何实现的呢，我们从声明变量开始吧~

```js
let price = 5
let quantity = 2
let total = price *  quantity // 10 ？
price = 20
console.log(`total is ${total}`) // 10

// 我们想要的 total 值希望是更新后的 40
```

## ⚠️ 问题

我们需要将 `total` 的计算过程存起来，这样我们就能够在 `price` 或者 `quantity` 变化时运行计算过程。

## ✅ 方案

首先，我们需要告诉应用程序，“这里有一个关于计算的方法，存起来，我会在数据更新的时候去运行它。“

![](https://cdn-images-1.medium.com/max/1600/0*Nh-FQoHiDHncmQSi.png)

我们创建一个记录函数并运行它：

```js
let price = 5
let quantity = 2
let total = 0
let target = null

target = () => { total = price * quantity }

record() // 记录我们想要运行的实例 
target() // 运行 total 计算过程
```

简单定义一个 `record`：

```js
let storage = [] // 将 target 函数存在这里
function record () { // target = () => { total = price * quantity }
	storage.push(target)
}
```

我们存储了 `target` 函数，我们需要运行它，需要顶一个 `replay` 函数来运行我们记录的函数：

```js
function replay () {
	storage.forEach( run => run() )
}
```

在代码中我们运行：

```js
price = 20
console.log(total) // => 10
replay()
console.log(total) // => 40
```

是不是足够的简单，代码可读性好，并且可以运行多次。FYI，这里用了最基础的方式进行编码，先扫盲。

```js
let price = 5
let quantity = 2
let total = 0
let target = null
let storage = []

target = () => { total = price * quantity }

function record () {
	storage.push(target)
}

function replay () {
	storage.forEach( run => run() )
}

record()
target()

price = 20
console.log(total) // => 10
replay()
console.log(total) // => 40
```

# 对象化
## ⚠️ 问题

我们可以不断记录我们需要的 `target`, 并进行 `record`，但是我们需要更健壮的模式去扩展我们的应用。也许面向对象的方式可以维护一个 targe 列表，我们用通知的方式进行回调。

## ✅ 方案 Dependency Class

我们通过一个属于自己的类进行行为的封装，一个标准的依赖类 **Dependency Class**，实现观察者模式。

如果我们想要创建一个管理依赖的类，标准模式如下：

```js
class Dep { // Stands for dependency 
	constructor () {
	  this.subscribers = [] // 依赖数组，当 notify() 调用时运行
	}

	depend () {
	  if (target && !this.subscribers.includes(target)) {
	    // target 存在并且不存在于依赖数组中，进行依赖注入
	    this.subscribers.push(target)
	  }
	}

	notify () { // 替代之前的 replay 函数
	  this.subscribers.forEach(sub => sub()) // 运行我们的 targets，或者观察者函数
  }
}
```

注意之前替换的方法，`storage` 替换成了构造函数中的 `subscribers`。`recod` 函数替换为 `depend`。`replay` 函数替换为 `notify`。

现在在运行：

```js
const dep = new Dep()

let price = 5
let quantity = 2
let total = 0
let target = () => { total = price * quantity }

dep.depend() // 依赖注入
target() // 计算 total

console.log(total) // => 10
price = 20
console.log(total) // => 10
dep.notify()
console.log(total) // => 40
```

工作正常，到这一步感觉奇怪的地方，配置 和 运行 `target` 的地方。 

# 观察者
## ⚠️ 问题

之后我们希望能够将 Dep 类应用在每一个变量中，然后优雅地通过匿名函数去观察更新。可能需要一个观察者 `watcher` 函数去满足这样的行为。

我们需要替换的代码：

```js
target = () => { total = price * quantity }
dep.depend()
target()
```

替换为：

```
watcher( () => {
  total = price * quantit
})
```

## ✅ 方案 A Watcher Function

我们先定义一个简单的 watcher 函数：

```js
function watcher (myFunc) {
	target = myFunc // 动态配置 target
	dep.depend() // 依赖注入
	target() // 回调 target 方法
  target = null // 重置 target
}
```

正如你所见， `watcher` 函数传入一个 `myFunc` 的形参，配置全局变量 `target`，调用 `dep.depend()` 进行依赖注入，回调 `target` 方法，最后，重置 `target`。

运行一下：

```js
price = 20
console.log(total) // => 10
dep.notify()
console.log(total) // => 40
```

你可能会质疑，为什么我们要对一个全局变量的 `target` 进行操作，这显得很傻，为什么不用参数传递进行操作呢？文章的最后将揭晓答案，答案也是显而易见的。

# 数据抽象
## ⚠️ 问题

我们现在有一个单一的 `Dep class`，但是我们真正想要的是我们每一个变量都拥有自己的 `Dep`。让我们先将数据抽象到 `properties`。

```js
let data = { price: 5, quantity: 2 }
```

将设每个属性都有自己的内置 Dep 类

![](https://cdn-images-1.medium.com/max/1600/0*kV4iCRoguwO5C_JQ.png)

然我我们运行：

```js
watcher( () => {
  total = data.price * data.quantit
})
```

当 `data.price` 的 value 开始存取时，我想让关于 `price` 属性的 Dep 类 push 我们的匿名函数（存储在 target 中）进入 subscriber 数组（通过调用 dep.depend()）。而当 `quantity` 的 value 开始存取时，我们也做同样的事情。

![](https://cdn-images-1.medium.com/max/1600/0*E-_YXfn3vJe7S_Ry.png)

如果我们有其他的匿名函数，假设存取了 `data.price`，同样的在 `price` 的 Dep 类中 push 此匿名函数。

![](https://cdn-images-1.medium.com/max/1600/0*wefv6my2WWLW2385.png)

当我们想通过 `dep.notify()`  进行 `price` 的依赖回调时候。我们想 在 `price` set 时候让回调执行。在最后我们要达到的效果是：

```
$ total
10
$ price = 20 // 回调 notify() 函数
$ total
40
```

## ✅ 方案 Object.defineProperty()

我们需要学习一下关于 [Object.defineProperty() - JavaScript | MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty)。它允许我们在 property 上定义 getter 和 setter 函数。我们展示一下最基本的用法：

```js
let data = { price: 5, quantity: 2 }

Object.defineProperty(data, 'price', { // 仅定义 price 属性
  get () { // 创建一个 get 方法
    console.log(`I was accessed`)
	},
	set (newVal) { // 创建一个 set 方法
    console.log(`I was changed`)
	}
})

data.price // 回调 get()
// => I was accessed
data.price = 20 // 回调 set()
// => I was changed
```

正如你所见，打印两行 log。然而，这并不能推翻既有功能， `get` 或者 `set` 任意的 `value`。`get()` 期望返回一个 value，`set()` 需要持续更新一个值，所以我们加入 `internalValue` 变量用于存储 `price` 的值。

```js
let data = { price: 5, quantity: 2 }

let internalValue = data.price // 初始值

Object.defineProperty(data, 'price', { // 仅定义 price 属性
  get () { // 创建一个 get 方法
    console.log(`Getting price: ${internalValue}`)
    return internalValue
	},
	set (newVal) { // 创建一个 set 方法
    console.log(`Setting price: ${newVal}`)
	  internalValue = newVal
	}
})

total = data.price * data.quantity // 回调 get()
// => Getting price: 5
data.price = 20 // 回调 set()
// => Setting price: 20
```

至此，我们有了一个当 get 或者 set 值的时候的通知方法。我们也可以用某种递归可以将此运行在我们的数据队列中？

FYI，`Object.keys(data)` 返回对象的 key 值列表。

```js
let data = { price: 5, quantity: 2 }

Object.keys(data).forEach(key => {
  let internalValue = data[key]
  Object.defineProperty(data, key, {
	  get () {
      console.log(`Getting ${key}: ${internalValue}`)
      return internalValue
	  },
	  set (newVal) {
      console.log(`Setting ${key}: ${newVal}`)
	    internalValue = newVal
	  }
  })
})

total = data.price * data.quantity
// => Getting price: 5
// => Getting quantity: 2
data.price = 20
// => Setting price: 20
```

# CI 集成
将所有的理念集成起来

```js
total = data.price * data.quantity
```

当代码碎片比如 get 函数的运行并且 get 到 `price` 的值，我们需要 `price` 记录在匿名函数 function(target) 中，如果 `price` 变化了，或者 **set** 了一个新的 value，会触发这个匿名函数并且 get return，它能够知道这里更新了一样。所以我们可以做如下抽象：

* **Get** => 记录这个匿名函数，如果值更新了，会运行此匿名函数

* **Set** => 运行保存的匿名函数，仅仅改变保存的值。

在我们的 Dep 类的实例中，抽象如下：

* **Price accessed (get)** => 回调 `dep.depend()` 去注入当前的 `target`

* **Price set** => 回调 price 绑定的 `dep.notify()` ，重新计算所有的 `targets`

让我们合并这两个理念，生成最终代码：

```js
let data = { price: 5, quantity: 2 }
let target = null

class Dep { // Stands for dependency 
	constructor () {
	  this.subscribers = [] // 依赖数组，当 notify() 调用时运行
	}

	depend () {
	  if (target && !this.subscribers.includes(target)) {
	    // target 存在并且不存在于依赖数组中，进行依赖注入
	    this.subscribers.push(target)
	  }
	}

	notify () { // 替代之前的 replay 函数
	  this.subscribers.forEach(sub => sub()) // 运行我们的 targets，或者观察者函数
  }
}

// 遍历数据的属性
Object.keys(data).forEach(key => {
  let internalValue = data[key]

	// 每个属性都有一个依赖类的实例
  const dep = new Dep()

  Object.defineProperty(data, key, {
	  get () {
      dep.depend()
      return internalValue
	  },
	  set (newVal) {
	    internalValue = newVal
		dep.notify()
	  }
  })
})

// watcher 不再调用 dep.depend
// 在数据 get 方法中运行
function watcher (myFunc) {
	target = myFunce
	target()
	target = null
}

watcher(()=> {
	data.total = data.price * data.quantity
})

data.total
// => 10
data.price = 20
// => 20
data.total
// => 40
data.quantity = 3
// => 3
data.total
// => 60
```

这已经达到了我们的期望，`price` 和 `quantity` 都成为响应式的数据了。

Vue 的数据响应图如下：

![](https://cdn-images-1.medium.com/max/1600/0*tB3MJCzh_cB6i3mS.png)

看到紫色的 Data 数据，里面的 getter 和 setter？是不是很熟悉了。每个组件的实例会有一个 `watcher`的实例（蓝色圆圈）, 从 getter 中收集依赖。然后 setter 会被回调，这里 **notifies** 通知 watcher 让组件重新渲染。下图是本例抽象的情况：

![](https://cdn-images-1.medium.com/max/1600/0*P268NBNs64Z-CERj.png)

显然，Vue 的内部转换相对于本例更复杂，但我们已经知道最基本的了。

# 知识点
* 如何创建一个 **Dep class** 并且注入所有的依赖执行（notify）
* 如何创建一个 **watcher** 进行代码管理，需要添加一个依赖项目
* 如何使用 **Object.defineProperty()** 创建一个 getters 和 setters












