title: js中几种位置和大小的理解

categories:
- Web开发
- JS
tags:
- Web开发
- JS

date: 2016/11/20
folder: web/js
en-title: js-size-and-position
---
在js的计算过程中，经常会用到元素的各种位置和大小信息，js中提供了多种方法。本文总结了js中经常容易混淆的位置和大小的概念，如clientWidth、offsetLeft、scrollLeft等等。
<!-- more -->
## 几种位置和宽高的理解

* js
  * clientLeft（元素左边框宽度）、clientTop（元素上边框的宽度）、clientWidth（元素内容宽度加上左右两端padding宽度，行内元素为0）、clientHeight（元素内容宽度加上上下两端padding宽度，行内元素为0）
  * offsetParent
    * 对于非定位元素是其根元素（标准模式下是html，怪异模式下是body）
    * 对于定位元素是其最近的定位父元素
  * offsetLeft（元素的外padding到offsetParent的外padding的距离。对于inline元素则是从外border到offsetParent的外padding）、offsetTop、offsetWidth（包含元素的宽度、padding以及滚动条（元素宽度会留一部分给滚动条）、offsetHeight
  * scrollLeft（元素内容向右滚动的距离）、scrollTop（元素内容向下滚动的距离）、scrollWidth（元素内容宽度与元素本身宽度的较大值，本身宽度包括padding）、scrollHeight
  * getBoundingClientRect（元素的宽高：除了margin以外的宽高。元素的位置：除了margin外元素的左上角与视口的左上角的相对位置。对于可滚动的元素，其内容在滚动过程中坐标会变化）
* jquery
  * position（相对于offset parent的位置）
  * scrollTop（与js中scrollTop相同）
  * width、height
    * 只包含内容宽度和高度，与`$().css('width')`返回的结果相同，只是类型是number，后者以'px'为单位。
  * offset（相对于`docuemnt`的位置，与scrollLeft、scrollTop相同）