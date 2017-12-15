title: 移动web开发中的viewport

categories:
- Web开发
- HTML
tags:
- Web开发
- HTML
- CSS

date: 2016/11/26
folder: web/html
en-title: meta-viewport
---
在对移动端做响应式布局时，一般都是直接对meta标签进行设置，然后通过媒体查询来对不同尺寸的设备进行样式的调整。但是为什么meta标签要这么设置，以及它与视口(viewport)有什么关系一直不是很理解，因此花了一点时间对这个问题进行了整理，也对其有了更深入的了解。
<!--more-->

## CSS中的像素和设备像素

css中的像素理解起来很简单，就是我们在css文件中设置的像素值，如`width: 500px`。而设备像素就是电脑或者手机屏幕的物理像素，而css像素和设备像素又有什么关系呢，这就涉及到一个属性`devicePixelRatio`，它是window对象下的属性，可以直接通过`window.devicePixelRatio`来读取它。对于一般情况下这个值都是1，在retina屏上则是2。这个属性表示的是**css中的一个像素对应的设备像素数**，因此一般情况下，css中的一个像素就对应设备上的一个像素，只有在某些特殊的显示屏上才会有css的一个像素对应多个像素的情况。

## 视口的概念

我们一般认为视口就是设备的可视区域，也就是用来显示网页的那一块区域。随着移动设备的出现，不同的设备可能有着不同的可视区域大小，具体可以通过[viewportsize](http://viewportsizes.com/)来查看，或者直接通过浏览器的开发工具来查看。

但在开发过程中会发现视口显示的像素并不是设备的可视区域的像素，举个例子：

```css
.container-1 {
    height: 300px;
   	background-color: #abccbb;
 }
 .contaienr-2 {
    height: 300px;
   	background-color: #343421;
 }
```

在iphone6的显示下效果如图：

![1](/images/viewport/1.png)

这里会看到，当两个container的宽度适应屏幕大小时显示的是980px，而不是设备的375px。不同的设备下这个值也不同，主要大小有980和1024，可以通过`document.documentElement.clientWidth`来得到。而至于这里为什么和设备像素不同，又会涉及到以下meta标签中viewport的知识。

## meta标签中的viewport

在做移动端web开发时都会对这个值进行设置，一般会设置成`<meta name="viewport" content="width=device-width, initial-scale=1.0, minimum-scale = 1.0, maximum-scale=1.0, user-scalable=0">`。这样设置会使得viewport宽度等于设备的宽度并且同时不允许用户进行手势缩放。

![2](/images/viewport/2.png)

这样再次查看上面这张图会发现它的视口宽度和设备宽度已经相等了。在开发过程中我们需要的就是这种效果。

在meta标签的viewport中，允许设置以下六种值：

* width：设置视口宽度，可以为具体的像素值或者是device-width
* height：设置视口的高度，一般不使用
* initial-scale：设置视口的初始缩放值
* minimum-scale：设置视口最大缩放值
* maximum-scale：设置视口最小缩放值
* user-scalable：设置用户是否可以手势缩放

因此这里可以看出，通过上面常用的meta viewport的设置可以使得视口宽度和设备宽度相等。这里要注意，**initial-scale、minimum-scale和maximum-scale指的是设备大小和视口大小的比值**。

现在来看图一的问题也就好理解了，iphone默认对视口进行了调整，即将视口调整到了980px，这里使用的是initial-scale来对其进行调整，可以计算出`375/980=0.38`即是这里的初始缩放值。

另一点需要注意的是既然width和user-scale都可以设置视口大小，那么两个同时使用则浏览器采用的是哪一个呢，经过测试最终效果使用的是两个中尺寸更大的那一个值。但两者有一个区别是：如果较大值是width，则无论设为多少，视口的大小就是这个值；但如果较大值是initial-scale，则视口的大小是有限制的，对于不同设备不全相同。

下面测试一下，不同情况下的显示效果：

css都设置为以下值：

```css
.container-1 {
    width: 1200px;
    height: 300px;
    background-color: #abccbb;
}
.container-2 {
    width: 800px;
    height: 300px;
    background-color: #343421;
}
```

如果没有设置meta viewport属性：

![3](/images/viewport/3.png)

可以看到，元素完全显示在屏幕当中，无横向滑动框，即可视区域大小为1200px。但这里注意body的大小被限制在了980px。可见视口大小还是980px。因此可以认为**meta viewport标签设置的尺寸就是视口大小，即文档尺寸大小**。而此时屏幕大小则是由浏览器自动计算缩放值来使得网页不会出现滚动条来得到的。

设置meta viewport属性为`<meta name='viewport' content='width=1000, initial-scale=1.0'>`:

![4](/images/viewport/4.png)

可以看到，文档宽度为1000px，超出屏幕宽度并出现滚动条。

## 总结

从以上过程可以看出，css的像素和设备像素概念上是不同的，在移动设备上不同设备的尺寸也不尽相同。为了能在开发时对设备像素和css像素进行统一，我们会采用通过`<meta name="viewport" content="width=device-width, initial-scale=1">`的方式来限制。对于width和initial-scale两个的区别，可以认为**initial-scale设置了当前屏幕可视区域下可以显示的尺寸，width则设置了html元素尺寸**。总的来说，在移动web开发中设置meta viewport则是为了让设备的视口大小和设备大小相等，从而方便通过根据不同设备的尺寸进行响应式设计。

