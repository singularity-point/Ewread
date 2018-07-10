# 稀疏数组真心话大冒险
#article

> [An adventure in sparse arrays](https://remysharp.com/2018/06/26/an-adventure-in-sparse-arrays) <br>
> [JavaScript 稀疏数组与孔(hole) - 简书](https://www.jianshu.com/p/ce776c0067fb)

## 概念
### 稀疏数组（sparse array）

稀疏数组与密集数组最大的不同，就是稀疏数组中可以有“孔”(hole)。
绝大多数 JavaScript 中的稀疏数组默认是都带孔的。（ES标准并没有这样规定）

### 孔（hole）

孔是逻辑上存在于数组中，但物理上不存在与内存中的那些数组项。在那些仅有少部分项被使用的数组中，孔可以大大减少内存空间的浪费。
孔更像是 `undefined` ，但并非真正的没有被定义，仅仅是数组中的孔没有被赋值。

## 简单实现（大冒险）

```js
new Array(1) // hole × 1
[ , ] // hole × 1
[ 1, ] // int(1), no holes, length: 1
[ 1, , ] // int(1) and hole × 1
```

上例中可以看出，数组逗号分隔符后边的元素完全被忽略了。

再看看容易混淆的情况：

```js
const a = [ undefined, , ];
console.log(a[0]) // undefined
console.log(a[1]) // undefined
```

以上数组的值都为 `undefined` ，但是又并非都为 `undefined`。可以看出，`a[0]` 的值是 `undefined`，`a[1]` 是一个孔。

我们用 `prop in object` 验证一下

```js
const a = [ undefined, , ];
console.log(0 in a) // true
console.log(1 in a) // false
```

这样看起来是不是很明朗了？但是真正的孔为 `undefined` 的真相是什么，我们继续。
使用原型链 `prototype` 进行一次赋值，来看看情况：

```js
// elsewhere in boobooland
Array.prototype[1] = 'fool!'
// and in my code
const a = [ undefined, , ];
console.log(0 in a) // true
console.log(1 in a) // true … 😱
```

那问题来了，如果原型链被污染了，这种方式验证孔的存在就显得不严谨了。当然使用 `proper` 的方式可以避免这种情况：

```js
// elsewhere in boobooland
Array.prototype[1] = 'fool!'
// and in my code
const a = [ undefined, , ];
console.log(a.hasOwnProperty(1)) // false
// 👊 have some of that boobooland
```

自从 `hasOwnProperty` 在 `ES3` 提出后，使用这个方法，我们可以称自己是一个 *古典编程者*。😂

## 为什么（真心话）

性能可以作为阐述孔的存在的原因，比如创建一个数组 `new Array(10000000)`，这里并没有 1 千万的值被分配在数组中，浏览器也不会存储任何数据。

### 在遍历中终止

我们验证一下在 `map`，`forEach` 和 `filter` 这3种遍历模式下的回调情况。

`forEach` 将跳过所有的孔，也会耗时。

```js
const a = new Array(100)
a[1] = 1
let i = 0
a.forEach(() => i++)
console.log(i) // 1 - the callback is never called
```

`map` 和 `forEach` 一样，跳过所有孔，但是也会在结果中 return 对应的孔。

```js
const a = [ undefined, , true ];
const res = a.map(v => typeof v);
console.log(res) // [ "undefined", hole × 1, "boolean" ]
```

当然，全是孔的数组使用 `map`  return 固定值，也不好使，仍然会 return 一个孔，对应的回调函数并不执行。

```js
new Array(10).map((_, i) => i + 1) // hole × 10
// not 1, 2, 3 … etc 😢
```

然后，`filter` 会移除对应的孔。

```js
const a = [ undefined, , true ];
const res = a.filter(() => true);
console.log(res) // [ undefined, true ] - no hole 🤔
```

### 在循环中进行

有两种循环可以遨游稀疏数组，第一种是经典的循环，`for`，当然，`while` 和 `do/while`一样起作用。

```js
const a = new Array(3);
for (let i = 0; i < a.length; i++) {
  console.log(i, a[i]) // logs 0…2 + undefined
}
```

另外，`ES6` 的数组方法，会将数组的孔转换为 `undefined`，这是因为在底层，数组展开使用了 [iterator 协议](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Iteration_protocols)。

这意味着在如果要拷贝数组的时候，需要谨慎使用 `...` 扩展运算，而使用 `slice` 方法拷贝。

```js
const a = [ 1, , ];
const b = [ ...a ];
const c = a.slice();

expect(a).toEqual(b); // false
expect(a).toEqual(c); // true
```

### 适用于稀疏数组的情况

对比一下 `Array.from({ length })` 的情况，可以看出耗时差距。

```js
const length = 1000000; // 1 million 🧐
let time = new Date().getTime()
new Array(length); // ~5ms
console.log(new Date().getTime() - time)
time = new Date().getTime()
Array.from({ length }) // ~150ms
console.log(new Date().getTime() - time)
```

## 总结
* 分隔符尾部元素会被忽略

```js
[ 1, 2, 3, ] // no hole at the end, just a regular trailing comma
```

* 分隔符中间为孔

```js
[ 1, , 2 ] // hole at index(1) aka empty
```

* 检测孔使用 `array.hasOwnProperty(index)`

```js
[ 1, , 2 ].hasOwnProperty(1) // false: index(1) does not exist, thus a hole
```

* 遍历方法，如 `map`，`forEach`，`every` 不会回调孔

```js
const a = new Array(1000);
let ctr = 0;
a.forEach(() => ctr++);
console.log(ctr); // 0 - the callback was never called
```

* `map` 返回的新数组会包含对应的孔

```js
[ 1, , 2 ].map(x => x * x) // [ 1, <empty>, 4 ]
```

* `filter` 返回的数组会移除孔

```js
[ 1, , 2 ].filter(x => true) // [ 1, 2 ]
```

* `keys` 和 `values` 返回的循环方法会遍历到孔，包含但不限于 `for key of array.keys())`

```js
const a = [ 'a', , 'b' ];
for (let [index, value] of a.entries()) {
  console.log(index, value);
}
/* logs:
- 0 'a'
- 1 undefined
- 2 'b'
*/
```

* 数组展开运算 `[...array]` 会转换孔为 `undefined`，这当然会增加内存消耗，影响性能

```js
[...[ 'a', , 'b' ]] // ['a', undefined, 'b']
```

* 庞大的数据创建非常迅速，比 `Array.from` 快多了

```js
const length = 10000000; // 10 million
new Array(length); // quick
Array.from({ length }) // less quick
```