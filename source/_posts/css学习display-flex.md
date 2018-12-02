---
title: 'css学习display:flex'
date: 2018-12-01 15:42:13
categories:

- 前端

tags:

- css

photos:
 - https://ws4.sinaimg.cn/large/006tNbRwly1fxskzky1xxj30au06z0uj.jpg

---
## 前言

标准盒模型的组成大家肯定都懂，由里向外content,padding,border,margin.
盒模型是有两种标准的，一个是标准模型，一个是IE模型。
![](https://ws2.sinaimg.cn/large/006tNbRwly1fxskzkr7ezj30ap075765.jpg)

![](https://ws4.sinaimg.cn/large/006tNbRwly1fxskzky1xxj30au06z0uj.jpg)
 从上面两图不难看出在标准模型中，盒模型的宽高只是内容（content）的宽高，
 而在IE模型中盒模型的宽高是内容(content)+填充(padding)+边框(border)的总宽高。

### css如何设置两种模型
这里用到了CSS3 的属性 box-sizing 

* 标准模型 
```css
box-sizing:content-box; 
```
* IE模型

```css
box-sizing:border-box; 
```
布局的传统解决方案，基于盒状模型，依赖 `display` 属性 +  `position`属性 + `float`属性。它对于那些特殊布局非常不方便，比如，垂直居中就不容易实现。
`display:flex` 弹性盒模型是css3新添加一个用于页面布局的全新CSS3模块功能。它可以把列表放在同一个方向（从左到右或从上到下排列），并且让这些列表能延伸到占用可用的空间。

## flex属性
在div这类块状元素元素设置`display:flex`或者给`span`这类内联元素设置`display:inline-flex`，flex布局即创建。Flex布局相关属性正好分为两拨，一拨作用在flex容器上，还有一拨作用在flex子项上。

### flex容器上
* `flex-direction`,
* `flex-wrap`,
* `flex-flow`,
* `justify-content`,
* `align-items`,
* `align-content`

###flex子项上
* `order`,
* `flex-grow`
* `flex-shrink`
* `flex-basis`
* `flex`
* `align-self`

本文dom样式结构

```html
<div class="box">
    <div class="item">1</div>
    <div class="item">2</div>
    <div class="item">3</div>
    <div class="item">4</div>
    <div class="item">5</div>
</div>
```

## 作用在flex容器上
### 1.  flex-direction:row | row-reverse | column | column-reverse;
> flex-direction用来控制子项整体布局方向，是从左往右还是从右往左，是从上往下还是从下往上

`row`是默认值，显示为行。方向为当前文档水平流方向，默认情况下是从左往右。如果当前水平文档流方向是rtl（如设置direction:rtl），则从右往左。
![](https://ws2.sinaimg.cn/large/006tNbRwly1fxsj7f6p1dj30t606omx6.jpg)
`row-reverse`显示为行。但方向和`row`属性值是反的。
![](https://ws4.sinaimg.cn/large/006tNbRwly1fxsjhxptzuj317204sdfv.jpg)
`column`显示为列。
![](https://ws1.sinaimg.cn/large/006tNbRwly1fxsjhygpxfj31740hwwex.jpg)
`column-reverse`显示为列。但方向和`column`属性值是反的。
![](https://ws2.sinaimg.cn/large/006tNbRwly1fxsjhz074hj316m0hmdg9.jpg)

### 2. flex-wrap:nowrap | wrap | wrap-reverse;
> `flex-wrap`用来控制子项整体单行显示还是换行显示，如果换行，则下面一行是否反方向显示。

`nowrap`默认值。默认值，表示单行显示，不换行。于是很容易出现宽度溢出的场景。
![](https://ws4.sinaimg.cn/large/006tNbRwly1fxsjhzedclj30t6054t9c.jpg)
`wrap`宽度不足换行显示。
![](https://ws1.sinaimg.cn/large/006tNbRwly1fxsjhzuqz9j30vy0860tc.jpg)
`wrap-reverse`宽度不足换行显示，但是是从下往上开始，也就是原本换行在下面的子项现在跑到上面。
![](https://ws2.sinaimg.cn/large/006tNbRwly1fxsji0kb06j30so07cwek.jpg)
### 3.  flex-flow:<‘flex-direction’> || <‘flex-wrap’>
> flex-flow属性是flex-direction和flex-wrap的缩写，表示flex布局的flow流动特性。例:flex-flow: row-reverse wrap-reverse;

### 4. justify-content:flex-start | flex-end | center | space-between | space-around | space-evenly
> justify-content属性决定了水平方向子项的对齐和分布方式

`flex-start`是默认值。逻辑CSS属性值，与文档流方向相关。默认表现为左对齐。
![](https://ws2.sinaimg.cn/large/006tNbRwly1fxsji1cd6aj30tc04g0sp.jpg)
`flex-end` 逻辑CSS属性值，与文档流方向相关。默认表现为右对齐。
![](https://ws1.sinaimg.cn/large/006tNbRwly1fxsji1rsjmj30ta056gll.jpg)
`center`表现为居中对齐。
![](https://ws1.sinaimg.cn/large/006tNbRwly1fxsji1rsjmj30ta056gll.jpg)
`space-between`表现为两端对齐。between是中间的意思，意思是多余的空白间距只在元素中间区域分配。使用抽象图形示意如下：
space-between分布效果示意
![](https://ws2.sinaimg.cn/large/006tNbRwly1fxsji2ncpsj30ti046mx4.jpg)
`space-around` around是环绕的意思，意思是每个flex子项两侧都环绕互不干扰的等宽的空白间距，最终视觉上边缘两侧的空白只有中间空白宽度一半。使用抽象图形示意如下：
space-around分布效果示意
![](https://ws2.sinaimg.cn/large/006tNbRwly1fxsji396j3j30to04q749.jpg)
`space-evenly`evenly是匀称、平等的意思。也就是视觉上，每个flex子项两侧空白间距完全相等。使用抽象图形
![](https://ws1.sinaimg.cn/large/006tNbRwly1fxsji3pq45j30tu04k0sp.jpg)

### 5. align-items: stretch | flex-start | flex-end | center | baseline;
> align-items中的items指的就是flex单行子项们，因此align-items指的就是flex子项们相对于flex容器在垂直方向上的对齐方式，大家是一起顶部对齐呢，底部对齐呢，还是拉伸对齐呢，类似这样。

`stretch`是默认值。flex子项拉伸。
![](https://ws1.sinaimg.cn/large/006tNbRwly1fxsji4iv1zj30tc06gjre.jpg)
`flex-start` 逻辑CSS属性值，与文档流方向相关。默认表现为容器顶部对齐。
![](https://ws4.sinaimg.cn/large/006tNbRwly1fxsji52n4wj30sy06mmx6.jpg)
`flex-end` 逻辑CSS属性值，与文档流方向相关。默认表现为容器底部对齐。
![](https://ws3.sinaimg.cn/large/006tNbRwly1fxsji5iin4j30sy0763yj.jpg)
` center` 表现为垂直居中对齐。
![](https://ws1.sinaimg.cn/large/006tNbRwly1fxsji6iza8j30sm070wei.jpg)
` baseline` 表现为所有flex子项都相对于flex容器的基线（字母x的下边缘）对齐。
![](https://ws1.sinaimg.cn/large/006tNbRwly1fxsji7f9z5j30t606omx6.jpg)
### 6.  align-content:stretch | flex-start | flex-end | center | space-between | space-around | space-evenly;
> align-content可以看成和justify-content是相似且对立的属性，justify-content指明水平方向flex子项的对齐和分布方式，而align-content则是指明垂直方向每一行flex元素的对齐和分布方式,控制多行。如果所有flex子项只有一行，则align-content属性是没有任何效果的。

`stretch`是默认值。每一行flex子元素都等比例拉伸。例如，如果共两行flex子元素，则每一行拉伸高度是50%。
![](https://ws1.sinaimg.cn/large/006tNbRwly1fxsji7txopj30sy0hit90.jpg)
`flex-start` 逻辑CSS属性值，与文档流方向相关。默认表现为顶部堆砌。
![](https://ws4.sinaimg.cn/large/006tNbRwly1fxsjvtfhjyj30te0ic3yc.jpg)
`flex-end`逻辑CSS属性值，与文档流方向相关。默认表现为底部堆放。
![](https://ws3.sinaimg.cn/large/006tNbRwly1fxsji8b4gij30tc0i6glx.jpg)
`center`表现为整体垂直居中对齐。
![](https://ws1.sinaimg.cn/large/006tNbRwly1fxsji8r8m4j30t00iedg5.jpg)
`space-between`表现为上下两行两端对齐。剩下每一行元素等分剩余空间。
![](https://ws1.sinaimg.cn/large/006tNbRwly1fxsji9pj2nj30u60i4mxh.jpg)
`space-around`每一行元素上下都享有独立不重叠的空白空间。
![](https://ws2.sinaimg.cn/large/006tNbRwly1fxsjia73qxj30sq0hq0t1.jpg)
`space-evenly`每一行元素都完全上下等分。
![](https://ws2.sinaimg.cn/large/006tNbRwly1fxsjvu7ua4j30ts0hw3yc.jpg)

## 作用在flex子容器上

### 1. order 
> 整数值，默认值是 0
> 我们可以通过设置order改变某一个flex子项的排序位置。

![](https://ws1.sinaimg.cn/large/006tNbRwly1fxsjiamobsj30u407k0sr.jpg)

### 2. flex-grow
> 整数值，默认值是 0
> flex-grow属性中的grow是扩展的意思，扩展的就是flex子项所占据的宽度，扩展所侵占的空间就是除去元素外的剩余的空白间隙。剩余空间除以设置所有子项flex-grow的值总和的值乘上子项flex-grow值就是该子项扩展的值;如果总和小于 1 时 除以 1 在乘子项flex-grow值;

![](https://ws3.sinaimg.cn/large/006tNbRwly1fxsjib3py5j30to06ydfu.jpg)


### 3. flex-shrink
> 整数值，默认值是 0 
>  shrink是“收缩”的意思，flex-shrink主要处理当flex容器空间不足时候，单个元素的收缩比例。。剩余空间除以设置所有子项flex-shrink的值总和的值乘上子项flex-grow值就是该子项缩放的值;如果总和小于 1 时 除以 1 在乘子项flex-shrink值;

![](https://ws4.sinaimg.cn/large/006tNbRwly1fxsjibkdbdj30iq046747.jpg)

### 4. flex-basis: | auto;
> 默认值是 auto 
> flex-basis定义了在分配剩余空间之前元素的默认大小。相当于对浏览器提前告知：浏览器兄，我要占据这么大的空间，提前帮我预留好。(默认大小)默认值是auto，就是自动。有设置width则占据空间就是width，没有设置就按内容宽度来。 如果同时设置width和flex-basis，就渲染表现来看，会忽略width。flex顾名思义就是弹性的意思，因此，实际上不建议对flex子项使用width属性，因为不够弹性。当剩余空间不足的时候，flex子项的实际宽度并通常不是设置的flex-basis尺寸，因为flex布局剩余空间不足的时候默认会收缩。

### 5. flex: none | auto | [ <'flex-grow'> <'flex-shrink'>? || <'flex-basis'> ]
> flex属性是flex-grow，flex-shrink和flex-basis的缩写。其中第2和第3个参数（flex-shrink和flex-basis）是可选的。默认值为0 1 auto。经过测试 flex默认值等同于flex:0 1 auto； flex:none等同于flex:0 0 auto；flex:auto等同于flex:1 1 auto;

### 6. align-self: auto | flex-start | flex-end | center | baseline | stretch;
> align-self指控制单独某一个flex子项的垂直对齐方式，写在flex容器上的这个align-items属性，后面是items，有个s，表示子项们，是全体；这里是self，单独一个个体。其他区别不大，语法几乎一样：唯一区别就是align-self多了个auto（默认值），表示继承自flex容器的align-items属性值。其他属性值含义一模一样

* flex-start
![](https://ws1.sinaimg.cn/large/006tNbRwly1fxsjicm9flj30us07cgln.jpg)

* flex-end
![](https://ws1.sinaimg.cn/large/006tNbRwly1fxsjvsrysdj30t207uq2q.jpg)

* center
![](https://ws3.sinaimg.cn/large/006tNbRwly1fxsjidk0m6j30tu07cgln.jpg)

* baseline
![](https://ws3.sinaimg.cn/large/006tNbRwly1fxsjidx10hj30tc06uwei.jpg)

* stretch
![](https://ws4.sinaimg.cn/large/006tNbRwly1fxsjiedp4vj30ss06st8q.jpg)


### 其他Flex知识点
1. 在Flex布局中，flex子元素的设置float，clear以及vertical-align属性都是没有用的。
2. Flex布局练习http://flexboxfroggy.com/