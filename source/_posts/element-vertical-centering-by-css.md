title: CSS实现元素的垂直居中

categories:
- Web开发
- CSS
tags:
- Web开发
- CSS

date: 2015/11/22
folder: web/css
en-title: element-vertical-centering-by-css
---

垂直居中指的是将元素在垂直方向上相对于父级元素达到一种居中的效果，在我们平时的布局中也会经常碰到垂直居中，在这里总结了下通过css实现垂直居中的各种常见方法，在使用各种方法时也要考虑到使用的场景。
<!-- more -->

以下css样式所应用到的html代码均如下:
```
<div class="content">
    <div class="box"></div>
</div>
```
要达到的效果是box相对于content实现垂直居中。

以下是各种实现方法总结：

## 1. 元素高度已知

1. 已知尺寸的块可以通过绝对定位和margin进行垂直居中
```
.content {
    display: relative;
 }

.content .box {
    display: absolute;
    height: 100px;
    top: 50%;
    margin-top: -50px;
}
```
    注意：通过这种方式元素被设置成了绝对定位，脱离了文档流，会对后面的元素位置产生影响。

2. 通过在box前设置一个浮动的空块来实现垂直居中
```
.floater {
    float: left;
    height: 50%;
    margin-bottom: -100px;
 }

.content .box {
    clear: left;
}
```
    注意：通过这种方式多用了一个空元素，同时因为使用了浮动元素所以应记得清除。

## 2. 元素高度未知

1. 通过table-cell元素的垂直居中属性实现
```
.content {
     display: table;
 }

 .content .box {
    display: table-cell;
    vertical-align: middle;
}
```
    或直接通过table布局，table-cell的vertical-align:middle默认属性，但两者具有区别。
2. 通过after或before伪类和vertical-align实现垂直居中
```
.content .box {
    width: auto;
    height: auto;
    display: inline-block;
    vertical-align: middle;
}

.content:after {
    content: "";
    height: 100%;
    width: 0;
    display: inline-block;
    vertical-align: middle;
}
```
    注意：定位元素需要设置为inline-block，同时都需要设置vertical-align:center属性。
3. 通过margin:auto自动填充外边距实现垂直居中
```
.content {
    position:relative;
 }

.content .box {
    position: absolute;
    height: 200px;
    top: 0;
    bottom: 0;
    margin-top: auto;
    margin-bottom: auto;
}
```
    注意：因为使用了绝对定位脱离了文档流，要考虑到对父元素和兄弟元素位置的影响。
4. 通过transform属性的translateY()实现垂直居中
```
.content {
    position: relative;
 }

 .content .box {
    position: absolute;
    top: 50%;
    transform: translateY(-50%);
 }
```
    注意：使用了绝对定位，元素脱离文档流。同时使用时注意兼容性，不支持IE9以下的浏览器，对于部分版本浏览器需加上-ms-、-webkit-等前缀。
5. 通过flexbox实现垂直居中
```
.content {
    display: flex;
    flex-direction: column;
    justify-content: center;
 }
```
    注意：弹性盒改变了块模型，它可以自动调整子元素使得定位子元素更加容易。它有自己的一些属性，使用了另一种不同的布局逻辑，会使部分元素属性失效，如float和vertical-align。

**小结：对于元素的垂直居中的情况判断一般是以该元素的高度是否已知，对于高度已知的情况，上述通过绝对定位和margin—top或通过增加一个浮动的空块均可解决，对于高度未知的情况，通过table布局、vartical-align属性、弹性盒、绝对定位与margin:auto的配合使用或是transfer属性都可实现垂直居中的布局。**

