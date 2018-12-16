---
title: js无限向上滚动
date: 2018-12-08 18:19:35
categories:

- 前端

tags:

- js
---
## 背景
需要做一个无限列表数据向上滚动的小需求。在页面插入大量dom 可能会造成页面的卡顿。页面用户体验效果不友好等问题
## 原理
1. 在页面插入两个父级dom 作为数据展示
2. 使用setInterval周期函数判断dom是否移动出展示区域
3. dom移出展示区域将dom移除，重新插入到展示元素尾部
4. dom未移出展示区域，让展示区域滚动条向下滚动

## 代码实现
```
  //数据展示区域
   <div class="list-content" id="demo">
    </div>
  <script>
  //目前假数据
  //设置定
  var arr = []
  for (var i = 0; i < 100; i++) {
     var item = {
          id: i,
          content: "恭喜" + i + "获得的了" + i + "元"
        }
        arr.push(item)
  }
  LoadMord("#demo", arr, 50, 6) 
 function LoadMord(str, arr, speed, len) {
        var s = speed || 50;    //滚动速度值，值越大速度越慢
        var a = arr || [];
        var l = len || 6;
        //插入dom
        var oDiv1 = $("<div id='demo1'></div>")
        var oDiv2 = $("<div id='demo2'></div>")
        var oDiv3 = $("<div id='demo3'></div>")
        for (var i = 0; i < l; i++) {
            var p = $("<p>" + a[i].content + "</p>")
            p.appendTo(oDiv1)
        }
        for (var i = 0; i < l; i++) {
            var p = $("<p>" + a[l + i].content + "</p>")
            p.appendTo(oDiv2)
        }
        oDiv1.appendTo($(str))
        oDiv2.appendTo($(str))
        var flag = 2
        function Marquee() {
            if ($("#demo1").position().top + $("#demo1")[0].offsetHeight == 0 || $("#demo2").position().top + $("#demo2")[0].offsetHeight == 0) {
                oDiv3.html("")
                for (var i = 0; i < l; i++) {
                    if (l * flag + i > a.length - 1) {
                        var p = $("<p></p>")
                    } else {
                        var p = $("<p>" + arr[l * flag + i].content + "</p>")
                    }
                    p.appendTo(oDiv3)
                }
                flag++
                if (oDiv3.html().length == 0) {
                    clearInterval(MyMar)
                }
            }
            if ($("#demo1").position().top + $("#demo1")[0].offsetHeight == 0) {
                oDiv1.remove()
                oDiv1.html(oDiv3.html())
                oDiv1.appendTo($(str))
            } else if ($("#demo2").position().top + $("#demo2")[0].offsetHeight == 0) {
                oDiv2.remove()
                oDiv2.html(oDiv3.html())
                oDiv2.appendTo($(str))
            }


            demo.scrollTop++
        }

        var MyMar = setInterval(Marquee, s);
    }
    </script>
```
## 问题总结
* 使用dom的offsetHeight和jquery.position()判断dom 是否移出展示区域
  未指定 offsetParent 导致获取到的 jquery.position().top 不准确
* dom 的scrollTop ++ 展示区域的dom向上滚动
  
##  搞清属性

### 关于offset
1. offsetParent
2. offsetTop    
3. offsetLeft
4. offsetWidth  
5. offsetHeight

其中offsetWidth与offsetHeight有个特点，就是这两个属性的值只与该元素有关，与周围元素（父级和子级元素无关）。然而，offsetLeft与offsetTop却不是这样，这两个属性与offsetParent有关。

`offsetParent` 如果当前元素的父级元素没有进行CSS定位（position为absolute或relative），offsetParent为body。如果当前元素的父级元素中有CSS定位（position为absolute或relative），offsetParent取最近的那个父级元素。

`offsetLeft`   等于(offsetParent的padding-left)+(中间元素的offsetWidth)+(当前元素的margin-left)。

`offsetTop`    等于(offsetParent的padding-top)+(中间元素的offsetHeight)+(当前元素的margin-top)。

`offsetWidth`  等于(border-width)*2+(padding-left)+(width)+(padding-right)

`offsetHeight` 等于(border-width)*2+(padding-top)+(height)+(padding-bottom)
 
`jquery.position()` 同样也是需要根据offsetParent获取值返回。获取匹配元素相对父元素的偏移。返回的对象包含两个整形属性：top 和 left。为精确计算结果，请在补白、边框和填充属性上使用像素单位。此方法只对可见元素有效。  
### 滚动轴
`scrollHeight`和 `scrollWidth`，可滚动的绝对宽高，包括隐藏不可见的部分（offset仅是相对于元素的width和height不包括隐藏部分）。
`scrollTop`和`scrollLeft`: 可滑动的元素（即元素出现滚动条的情况时）内部在xy轴上滑动的距离，可为其赋值。
### 元素本身
`clientHeight`和`clientWidth`：可视区域的宽高（不同浏览器中clientHeight和offsetWidth有区别）。
`clientTop`和`clientLeft`:表示内容区域的左，上角相对于整个元素左，上角的位置（包括边框）
## 参考
[CSSOM视图模式(CSSOM View Module)相关整理](https://www.zhangxinxu.com/wordpress/2011/09/cssom%E8%A7%86%E5%9B%BE%E6%A8%A1%E5%BC%8Fcssom-view-module%E7%9B%B8%E5%85%B3%E6%95%B4%E7%90%86%E4%B8%8E%E4%BB%8B%E7%BB%8D/)