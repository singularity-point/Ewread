### 文章

推荐文章：[5 common practices that you can stop doing in React](https://blog.logrocket.com/5-common-practices-that-you-can-stop-doing-in-react-9e866df5d269)

### 总结

文章主要内容：

1.  没有必要从项目一开始就想着优化 React，因为 React 本身是基于 virtual DOM 的，如果不是遇到性能问题，没有必要一开始就优化性能。
2.  如果使用服务端渲染仅仅是为了 SEO，那大可不必，因为在 2015 年 10 月的时候 Google 就支持了抓取 CSS 文件和 JS 文件。
3.  使用内联 CSS 而不是 import CSS 文件，应该使用 CSS in JS，推荐使用 styled-components，至于为什么我没看出来 😂。
4.  React 中三元表达式比较好用，但是如果嵌套太多会导致代码很难看，推荐使用 babel 插件——[JSX Control Statements](https://github.com/AlexGilleran/jsx-control-statements#readme)，来扩展 JSX，关于更多的 React 条件渲染可以阅读文章——[8 methods for conditional rendering in React](https://blog.logrocket.com/conditional-rendering-in-react-c6b0e5af381e)。
5.  在 React render 中最好不要使用闭包，要在类中定义一个方法代替。

render 中使用了闭包

```js
class SayHi extends Component {

render () {
 return () {
  <Button onClick={(e) => console.log('Say Hi', e)}>
    Click Me
  </Button>
 }
}
}
```

类中定义一个方法代码闭包

```js
class SayHi extends Component {

  showHiMessage = this.showMessage('Hi')

  render () {
   return () {
      <Button onClick={this.showHiMessage}>
            Click Me
      </Button>
   }
  }
}
```
