title: d3.js源码分析——Path

categories:
- Web开发
- JS
tags:
- Web开发
- D3.js

date: 2016/09/02
folder: web/js
en-title: d3js-source-code-path
---
d3的path部分是对原生绘制方法的一种封装形式。由于d3.js内部是以svg作为默认的绘图方式，因此内部的计算方式都是将数据转换成svg中`path`元素的`d`属性值。通过统一的接口让其和canvas绘图的api保持一致。
<!--more-->

## Path
d3的path部分是为了模拟`canvas`的绘图方式，但是采用的是svg来作图。源码如下：
```
//path构造函数
function Path() {
    this._x0 = this._y0 = // 当前路径的起点
    this._x1 = this._y1 = null; // 当前路径的终点
    this._ = [];
}

function path() {
    return new Path;
}

Path.prototype = path.prototype = {
    constructor: Path,
    //移动至指定位置
    moveTo: function(x, y) {
      this._.push("M", this._x0 = this._x1 = +x, ",", this._y0 = this._y1 = +y);
    },
    //关闭路径
    closePath: function() {
      if (this._x1 !== null) {
        this._x1 = this._x0, this._y1 = this._y0;
        this._.push("Z");
      }
    },
    //绘制直线
    lineTo: function(x, y) {
      this._.push("L", this._x1 = +x, ",", this._y1 = +y);
    },
    //绘制二次贝塞尔曲线
    quadraticCurveTo: function(x1, y1, x, y) {
      // Q x1 x2（控制点）, x y（终点）
      this._.push("Q", +x1, ",", +y1, ",", this._x1 = +x, ",", this._y1 = +y);
    },
    //绘制三次贝塞尔曲线
    bezierCurveTo: function(x1, y1, x2, y2, x, y) {
      this._.push("C", +x1, ",", +y1, ",", +x2, ",", +y2, ",", this._x1 = +x, ",", this._y1 = +y);
    },
    arcTo: function(x1, y1, x2, y2, r) {
      x1 = +x1, y1 = +y1, x2 = +x2, y2 = +y2, r = +r;
      var x0 = this._x1,
          y0 = this._y1,
          x21 = x2 - x1,
          y21 = y2 - y1,
          x01 = x0 - x1,
          y01 = y0 - y1,
          l01_2 = x01 * x01 + y01 * y01;

      // Is the radius negative? Error.
      if (r < 0) throw new Error("negative radius: " + r);

      // Is this path empty? Move to (x1,y1).
      if (this._x1 === null) {
        this._.push(
          "M", this._x1 = x1, ",", this._y1 = y1
        );
      }

      // Or, is (x1,y1) coincident with (x0,y0)? Do nothing.
      else if (!(l01_2 > epsilon));

      // Or, are (x0,y0), (x1,y1) and (x2,y2) collinear?
      // Equivalently, is (x1,y1) coincident with (x2,y2)?
      // Or, is the radius zero? Line to (x1,y1).
      else if (!(Math.abs(y01 * x21 - y21 * x01) > epsilon) || !r) {
        this._.push(
          "L", this._x1 = x1, ",", this._y1 = y1
        );
      }

      // Otherwise, draw an arc!
      else {
        var x20 = x2 - x0,
            y20 = y2 - y0,
            l21_2 = x21 * x21 + y21 * y21,
            l20_2 = x20 * x20 + y20 * y20,
            l21 = Math.sqrt(l21_2),
            l01 = Math.sqrt(l01_2),
            l = r * Math.tan((pi$1 - Math.acos((l21_2 + l01_2 - l20_2) / (2 * l21 * l01))) / 2),
            t01 = l / l01,
            t21 = l / l21;

        // If the start tangent is not coincident with (x0,y0), line to.
        if (Math.abs(t01 - 1) > epsilon) {
          this._.push(
            "L", x1 + t01 * x01, ",", y1 + t01 * y01
          );
        }

        this._.push(
          "A", r, ",", r, ",0,0,", +(y01 * x20 > x01 * y20), ",", this._x1 = x1 + t21 * x21, ",", this._y1 = y1 + t21 * y21
        );
      }
    },
    //(x, y)为参照点坐标，a0和a1分别为起点弧度和终点弧度
    arc: function(x, y, r, a0, a1, ccw) {
      x = +x, y = +y, r = +r;
      var dx = r * Math.cos(a0),
          dy = r * Math.sin(a0),
          //起点(x0, y0)坐标
          x0 = x + dx,
          y0 = y + dy,
          //clockwise，顺时针
          cw = 1 ^ ccw,
          da = ccw ? a0 - a1 : a1 - a0;

      if (r < 0) throw new Error("negative radius: " + r);

      // 如果path为空，则move到(x0, y0)
      if (this._x1 === null) {
        this._.push(
          "M", x0, ",", y0
        );
      }

      // (x0, y0)与之前位置不一致，用直线连接到(x0, y0)
      else if (Math.abs(this._x1 - x0) > epsilon || Math.abs(this._y1 - y0) > epsilon) {
        this._.push(
          "L", x0, ",", y0
        );
      }

      if (!r) return;

      // 如果是一个圆，直接画两个圆弧实现这个圆形
      // A rx（x轴半径） ry（y轴半径） x-axis-rotation（x轴逆时针旋转角度） large-arc-flag（0表示小角度弧即小于180°，1表示大角度弧） sweep-flag（0表示从起点到终点逆时针画弧，1表示顺时针） x（弧线终点x轴） y（弧线终点y轴）
      if (da > tauEpsilon) {
        this._.push(
          "A", r, ",", r, ",0,1,", cw, ",", x - dx, ",", y - dy,
          "A", r, ",", r, ",0,1,", cw, ",", this._x1 = x0, ",", this._y1 = y0
        );
      }

      // 以r为半径，从当前点向endAngle位置画弧线
      else {
        if (da < 0) da = da % tau$1 + tau$1;
        this._.push(
          "A", r, ",", r, ",0,", +(da >= pi$1), ",", cw, ",", this._x1 = x + r * Math.cos(a1), ",", this._y1 = y + r * Math.sin(a1)
        );
      }
    },
    rect: function(x, y, w, h) {
      this._.push("M", this._x0 = this._x1 = +x, ",", this._y0 = this._y1 = +y, "h", +w, "v", +h, "h", -w, "Z");
    },
    // 将整个数组转化成一个字符串
    toString: function() {
      return this._.join("");
    }
  };
```




