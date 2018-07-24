# Css ‘ch’ 字符宽度单位
#article


> [What is the CSS ‘ch’ Unit?  –  Eric’s Archived Thoughts](https://meyerweb.com/eric/thoughts/2018/06/28/what-is-the-css-ch-unit/?utm_source=CSS-Weekly&utm_campaign=Issue-321&utm_medium=web) 

# 概念
`ch` 单位字面意思为 “character with”，这表明我们可以进行如下操作：

* 每一行保证最大 60 个字符的宽度
* 定义一个图片保持一个确定的字符宽度

但是有个问题，如何确认标准单位长度，是和固定的字体有关系么？答案是否定的，我们看一下官方[定义](https://drafts.csswg.org/css-values-3/#ch)：

```
Equal to the used advance measure of the "0" (ZERO, U+0030) glyph found in the font used to render it. (The advance measure of a glyph is its advance width or height, whichever is in the inline axis of the element.)
```

也就是说，这个是以字符 `0` 的宽度作为标准单位，包含影响该字符的 CSS 属性。

# 兼容性
> [Can I use… Support tables for HTML5, CSS3, etc](https://caniuse.com/#search=ch)

出了 Opera Mini 浏览器不兼容，其他都还好。（😂 感觉残疾了）

不过 关于 Opera Mini，有如下声明：

```
*In most cases Opera Mini 
processing is done via Opera servers, which often prevents JS from working correctly.
As of version 8, the default mode of Opera Mini on iOS uses the iOS Safari engine, though Mini mode can be enabled.
```

看来补救的挺好~ 那我们可以尝试一下

# 应用
> [A Pen by  chenzehui](https://codepen.io/Sariel/pen/LBGrmY)

`Courier` 字体下并无差异，所有英文字符都拥有相同的宽度。其他字体，都遵循 `0` 字符宽度的基础规则。

所以，使用 `ch` 可用作一个近似值，大部分情况下字体差异还是有的，根据不同字体，限制容器宽度。

比如一个 80 字符宽度的容器，可能具体的值需要设置成 `60ch`，所以使用时稍加注意就好~

能够解决前端的字符限制，图片与适配字符的情况。






