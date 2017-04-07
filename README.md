网页性能管理详解
==============

你遇到过性能差的网页吗？

这种网页相应非常慢，占用大量的cpu和内存，浏览起来非常卡顿，页面的动画效果也不流畅。

你会有什么反应》我猜想，大多数用户会关闭这个页面，改为访问其他网站。作为一个开发者，肯定不愿意看到这种情况，那么怎样才能提高性能呢？

**本文将详细介绍性能问题的出现原因，以及结局方法**

## 一、网页生成的过程

要理解网页性能为什么不好，就要了解网页是怎么生成的。

![](images/1.png)

网页的生成过程，大致分为五步：

1.	html代码转化dom
2.	css代码转化成cssdom（CSS Object Model）
3.	结合DOM和CSSDOM，生成一颗渲染树（包含每个节点的视觉信息）
4.	生成布局（layout），即将所有渲染树的所有节点进行平面合成
5.	将布局绘制（paint）在屏幕上

这五步，第一步到第三步都非常快，耗时的是第四步和第五步。

**“生成布局（flow）和绘制（paint）这两部，合成为渲染（render）”**

![](images/2.png)


## 二、重排和重绘

**网页生成的时候，至少会渲染一次。用户访问的过程中，还会不断重新渲染**。

一下三种情况，会导致网页重新渲染

*	修改DOM
*	修改样式表
*	用户事件（比如鼠标悬停、页面滚动、输入框键入文字、改变窗口大小等等）

**重新渲染，就需要重新生成布局和重新绘制。前置叫做“重排”（reflow），后者叫“重绘”（repaint）**

需要注意的是“重绘”不一定需要“重排”，比如改变某个网页元素的颜色，就只会触发“重绘”，不会触发“重排”，因为布局没有改变。但是“重排”必然导致“重绘”，比如改变一个网页元素的位置，就会同时出发“重绘”和“重排”，因为布局改变了。

## 三、对于性能的影响

重排和重绘不断触发，这是不可避免的。但是，他们非常耗费资源，是导致网页性能低下的根本原因。

**提高网页性能，就是降低“重排”和“重绘”的频率和成本，尽量减少触发重新渲染**。

前面提到，DOM变动和样式变动，都会触发重新渲染。但是，浏览器已经很智能了尽量把所有的变动集中在一起，排成一个队列，然后一次性执行，精良避免多次重新渲染。

```js

div.style.color = 'blue';
div.style.marginTop = '30px';

```

上面的代码中，div元素有两个样式变动，但是浏览器只会触发一次重排和重绘。

如果写的不好，就会触发两次重排和重绘。

```js

div.style.color = 'blue';
var margin = parseint(div.style.marginTop);
div.style.marginTop = (margin + 10) + 'px';

```

上面的代码对div元素设置背景色以后，第二行要求浏览器给出该元素的位置，所以浏览器不得不立即重排。

一般来说，样式的写操作之后，如果有下面这些属性的读操作，都会引发浏览器立即重新渲染。

*	offsetTop/offsetLeft/offsetWidth/offsetHeight
*	scrollTop/scrollLeft/scrollWidth/scrollHeight
*	clientTop/clientLeft/clientWidth/clientHeight
*	getComputedStyle()

所以，从性能角度考虑，尽量不要把读操作和写操作，放在一个语句里面。

```js


// bad
div.style.left = div.offsetLeft + 10 + "px";
div.style.top = div.offsetTop + 10 + "px";

// good
var left = div.offsetLeft;
var top  = div.offsetTop;
div.style.left = left + 10 + "px";
div.style.top = top + 10 + "px";

```

一般的规则是：

*	样式表越简单，重排和重绘就越快
*	重排和重绘的DOM元素层级越高，成本就越高
*	table元素的重排和重绘成本，要高于div元素

## 四、提高性能的九个小技巧

有一些技巧可以降低浏览器重新渲染的频率和成本。

**第一条**是上一节说到的，DOM的多个读操作（或者写操作），应该放在一起。不要两个读操作之间，加入一个写操作。

**第二条**，如果某个样式是通过重排得到的，那么最好缓存结果。避免下一次用到的时候，浏览器又要重排。

**第三条**，不要一条条地改变样式，而要通过改变class，或者csstext属性，一次性地改变样式。

```js

// bad
var left = 10;
var top = 10;
el.style.left = left + "px";
el.style.top  = top  + "px";

// good 
el.className += " theclassname";

// good
el.style.cssText += "; left: " + left + "px; top: " + top + "px;";

```

**第四条**，尽量使用离线DOM，而不是真实的网面DOM，来改变元素样式。比如，操作Doucument Fragment对象，完成后再把这个对象加入DOM。再比如，使用cloneNode()方法，在科隆的节点上操作，然后再用科隆的节点替换原始节点。

**第五条**，现将元素设置为`display:none`（需要一次重排和重绘），然后对这个节点进行100次操作，最后再回复显示（需要一次重排和重绘）。这样一来，你就用两次重新渲染，取代了可能高达100次的重新渲染。

**第六条**,position属性`absolute`或`fixed`的元素，重排的开销会比较小，因为不用考虑它对其他元素的影响。

**第七条**，只在必要的时候，才将元素的display属性为可见，因为不可见的元素不影响重排和重绘。另外，`visibility:hidden`的元素只对重绘有影响，不影响重排。

**第八条**，使用虚拟DOM的脚本库，比如React等。

**第九条**，使用window.requestAnimationFrame()、window.requestIdleCallback()这两个方法调节重新渲染（详见后文）

## 五、刷新率

未完待续[http://www.ruanyifeng.com/blog/2015/09/web-page-performance-in-depth.html](http://www.ruanyifeng.com/blog/2015/09/web-page-performance-in-depth.html)
