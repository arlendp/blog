title: d3.js源码分析——Shape

categories:
- Web开发
- JS
tags:
- Web开发
- D3.js

date: 2016/09/02
folder: web/js
en-title: d3js-source-code-shape
---
shape模块提供各种形状的生成器，这些形状的产生是数据驱动的，通过控制输入数据来形成一种视觉的表现。
<!--more-->

## Pies
饼图生成器不直接产生图形，而是计算出需要的角度信息，然后传入`d3.arc`中进行绘制。
```
  // 绘制饼图
function pie() {
    var value = identity$1,
        sortValues = descending$1,
        sort = null,
        startAngle = constant$1(0),
        endAngle = constant$1(tau$2),
        padAngle = constant$1(0);

    function pie(data) {
      var i,
          n = data.length,
          j,
          k,
          //统计data数组中的数据和
          sum = 0,
          index = new Array(n),
          arcs = new Array(n),
          a0 = +startAngle.apply(this, arguments),
          //将|endAngle - startAngle|限定在 2 * PI之间
          da = Math.min(tau$2, Math.max(-tau$2, endAngle.apply(this, arguments) - a0)),
          a1,
          //限定padAngle的范围
          p = Math.min(Math.abs(da) / n, padAngle.apply(this, arguments)),
          // da<0表示弧线为逆时针方向，pa的值也应进行相应处理
          pa = p * (da < 0 ? -1 : 1),
          v;

      for (i = 0; i < n; ++i) {
        if ((v = arcs[index[i] = i] = +value(data[i], i, data)) > 0) {
          sum += v;
        }
      }

      // 按照处理后的arcs数据大小对index进行排序，或者直接对data进行排序
      if (sortValues != null) index.sort(function(i, j) { return sortValues(arcs[i], arcs[j]); });
      else if (sort != null) index.sort(function(i, j) { return sort(data[i], data[j]); });

      // 计算arcs，按照排序后的index来逐个计算
      for (i = 0, k = sum ? (da - n * pa) / sum : 0; i < n; ++i, a0 = a1) {
        j = index[i], v = arcs[j], a1 = a0 + (v > 0 ? v * k : 0) + pa, arcs[j] = {
          data: data[j],
          index: i,
          value: v,
          startAngle: a0,
          endAngle: a1,
          padAngle: p
        };
      }

      return arcs;
    }
    //设置value函数或数值，value函数会被依次传入data[i]、i和data。
    pie.value = function(_) {
      return arguments.length ? (value = typeof _ === "function" ? _ : constant$1(+_), pie) : value;
    };
    //设置数值的排序方式
    pie.sortValues = function(_) {
      return arguments.length ? (sortValues = _, sort = null, pie) : sortValues;
    };

    pie.sort = function(_) {
      return arguments.length ? (sort = _, sortValues = null, pie) : sort;
    };

    pie.startAngle = function(_) {
      return arguments.length ? (startAngle = typeof _ === "function" ? _ : constant$1(+_), pie) : startAngle;
    };

    pie.endAngle = function(_) {
      return arguments.length ? (endAngle = typeof _ === "function" ? _ : constant$1(+_), pie) : endAngle;
    };

    pie.padAngle = function(_) {
      return arguments.length ? (padAngle = typeof _ === "function" ? _ : constant$1(+_), pie) : padAngle;
    };

    return pie;
}
```

## Lines
可以产生样条曲线或者多段线。

### d3.line
默认的设置是构造多条直线段。
```
//d3.line()，绘制多条直线段，中间可能断开
  function line() {
    var x$$ = x,
        y$$ = y,
        defined = constant$1(true),
        context = null,
        curve = curveLinear,
        output = null;

    function line(data) {
      var i,
          n = data.length,
          d,
          defined0 = false,
          buffer;

      if (context == null) output = curve(buffer = path());

      for (i = 0; i <= n; ++i) {
        if (!(i < n && defined(d = data[i], i, data)) === defined0) {
          if (defined0 = !defined0) output.lineStart();
          else output.lineEnd();
        }
        if (defined0) output.point(+x$$(d, i, data), +y$$(d, i, data));
      }
    // 返回path的计算结果
      if (buffer) return output = null, buffer + "" || null;
    }
    // 设置获取x的函数
    line.x = function(_) {
      return arguments.length ? (x$$ = typeof _ === "function" ? _ : constant$1(+_), line) : x$$;
    };
    // 设置获取y的函数
    line.y = function(_) {
      return arguments.length ? (y$$ = typeof _ === "function" ? _ : constant$1(+_), line) : y$$;
    };
    // defined函数用于判断当前点是否已被定义，若为true，则会计算x、y坐标和绘制直线；否则会跳过当前点，结束当前直线的绘制。
    line.defined = function(_) {
      return arguments.length ? (defined = typeof _ === "function" ? _ : constant$1(!!_), line) : defined;
    };
    // 设置curve函数
    line.curve = function(_) {
      return arguments.length ? (curve = _, context != null && (output = curve(context)), line) : curve;
    };
    // 设置context
    line.context = function(_) {
      return arguments.length ? (_ == null ? context = output = null : output = curve(context = _), line) : context;
    };

    return line;
}
```

### d3.radialLine
构造放射线，与上述`d3.line`类似，只是将x、y函数替换成角度和半径函数，并且改放射线总是相对于(0, 0)进行绘制。
```
// d3.radialLine，在d3.line的基础上进行修改，主要的差别是curve函数和坐标系。
function radialLine$1() {
    return radialLine(line().curve(curveRadialLinear));
}

// 构造放射线，分别构造线性和放射状曲线
var curveRadialLinear = curveRadial(curveLinear);

//线性的curve函数
function curveLinear(context) {
    return new Linear(context);
}

//将当前curve函数包装成放射状curve
function curveRadial(curve) {
    function radial(context) {
      return new Radial(curve(context));
    }
    radial._curve = curve;
    return radial;
}

// 将line中的(x, y)坐标替换成(angle, radius)
function radialLine(l) {
    var c = l.curve;

    l.angle = l.x, delete l.x;
    l.radius = l.y, delete l.y;

    l.curve = function(_) {
      return arguments.length ? c(curveRadial(_)) : c()._curve;
    };
    return l;
}
```

## Areas
用于产生一块区域。
### d3.area
```
/* d3.area
 * 首先根据x1, y1函数进行绘制，完成后根据x0, y0函数来反方向绘制，整个图形的绘制过程是顺时针方向。
 * 默认绘制的是x0 = x1，y0 = 0的一块区域。
 */
 function area$1() {
    var x0 = x,
        x1 = null,
        y0 = constant$1(0),
        y1 = y,
        defined = constant$1(true),
        context = null,
        curve = curveLinear,
        output = null;

    function area(data) {
      var i,
          j,
          k,
          n = data.length,
          d,
          defined0 = false,
          buffer,
          x0z = new Array(n),
          y0z = new Array(n);

      if (context == null) output = curve(buffer = path());

      for (i = 0; i <= n; ++i) {
        if (!(i < n && defined(d = data[i], i, data)) === defined0) {
          //defined0由false变为true时，表示绘制开始，相反表示绘制结束
          if (defined0 = !defined0) {
            j = i;
            output.areaStart();
            output.lineStart();
          } else {
            output.lineEnd();
            output.lineStart();
            //反向绘制x0z, y0z，绘制方向为顺时针方向
            for (k = i - 1; k >= j; --k) {
              output.point(x0z[k], y0z[k]);
            }
            output.lineEnd();
            // 关闭绘制区域
            output.areaEnd();
          }
        }
        //defined0为true时可以绘制
        if (defined0) {
          x0z[i] = +x0(d, i, data), y0z[i] = +y0(d, i, data);
          //优先使用x1和y1函数进行计算
          output.point(x1 ? +x1(d, i, data) : x0z[i], y1 ? +y1(d, i, data) : y0z[i]);
        }
      }

      if (buffer) return output = null, buffer + "" || null;
    }
    // 返回与当前area有相同defined、curve和context的line构造器
    function arealine() {
      return line().defined(defined).curve(curve).context(context);
    }
    // 设置x函数，将该函数赋值给x0，null赋值给x1
    area.x = function(_) {
      return arguments.length ? (x0 = typeof _ === "function" ? _ : constant$1(+_), x1 = null, area) : x0;
    };

    area.x0 = function(_) {
      return arguments.length ? (x0 = typeof _ === "function" ? _ : constant$1(+_), area) : x0;
    };

    area.x1 = function(_) {
      return arguments.length ? (x1 = _ == null ? null : typeof _ === "function" ? _ : constant$1(+_), area) : x1;
    };
    // 设置y函数，将该函数赋值给y0，null赋值给y1
    area.y = function(_) {
      return arguments.length ? (y0 = typeof _ === "function" ? _ : constant$1(+_), y1 = null, area) : y0;
    };

    area.y0 = function(_) {
      return arguments.length ? (y0 = typeof _ === "function" ? _ : constant$1(+_), area) : y0;
    };

    area.y1 = function(_) {
      return arguments.length ? (y1 = _ == null ? null : typeof _ === "function" ? _ : constant$1(+_), area) : y1;
    };
    // 分别对line构造器设置x和y函数
    area.lineX0 =
    area.lineY0 = function() {
      return arealine().x(x0).y(y0);
    };

    area.lineY1 = function() {
      return arealine().x(x0).y(y1);
    };

    area.lineX1 = function() {
      return arealine().x(x1).y(y0);
    };
    // defined函数用来判断是否绘制当前点。这样可以生成离散的图形。
    area.defined = function(_) {
      return arguments.length ? (defined = typeof _ === "function" ? _ : constant$1(!!_), area) : defined;
    };

    area.curve = function(_) {
      return arguments.length ? (curve = _, context != null && (output = curve(context)), area) : curve;
    };

    area.context = function(_) {
      return arguments.length ? (_ == null ? context = output = null : output = curve(context = _), area) : context;
    };

    return area;
  }
```

### d3.radialArea
绘制放射状区域。
```
/*
 * d3.radialArea
 * 在area的基础上修改curve函数，将x,y坐标转换为angle,radius坐标，其余绘制方式不变
 */
function radialArea() {
    var a = area$1().curve(curveRadialLinear),
        c = a.curve,
        x0 = a.lineX0,
        x1 = a.lineX1,
        y0 = a.lineY0,
        y1 = a.lineY1;
    // 将d3.area中的(x0, y0)和(x1, y1)转化成(startAngle, innerRadius)和(endAngle, outerRadius)
    a.angle = a.x, delete a.x;
    a.startAngle = a.x0, delete a.x0;
    a.endAngle = a.x1, delete a.x1;
    a.radius = a.y, delete a.y;
    a.innerRadius = a.y0, delete a.y0;
    a.outerRadius = a.y1, delete a.y1;
    a.lineStartAngle = function() { return radialLine(x0()); }, delete a.lineX0;
    a.lineEndAngle = function() { return radialLine(x1()); }, delete a.lineX1;
    a.lineInnerRadius = function() { return radialLine(y0()); }, delete a.lineY0;
    a.lineOuterRadius = function() { return radialLine(y1()); }, delete a.lineY1;
    // 对自定义的curve函数进行包装，防止计算时方法不能使用
    a.curve = function(_) {
      return arguments.length ? c(curveRadial(_)) : c()._curve;
    };
    return a;
  }
```

## Curve
curve的功能就是将离散的点进行连接，形成一个连续的图形，它并不是直接使用，而是传入上述如`d3.line`、`d3.area`等函数的`curve`函数中来控制这些离散的点的连接方式。

### d3.curveBasis
通过特定控制点的贝塞尔曲线将离散的点进行连接。
```
function point(that, x, y) {
    that._context.bezierCurveTo(
      (2 * that._x0 + that._x1) / 3,
      (2 * that._y0 + that._y1) / 3,
      (that._x0 + 2 * that._x1) / 3,
      (that._y0 + 2 * that._y1) / 3,
      (that._x0 + 4 * that._x1 + x) / 6,
      (that._y0 + 4 * that._y1 + y) / 6
    );
}
function Basis(context) {
    this._context = context;
}

Basis.prototype = {
    areaStart: function() {
      this._line = 0;
    },
    areaEnd: function() {
      this._line = NaN;
    },
    lineStart: function() {
      this._x0 = this._x1 =
      this._y0 = this._y1 = NaN;
      this._point = 0;
    },
    lineEnd: function() {
      switch (this._point) {
        case 3: point(this, this._x1, this._y1); // proceed
        case 2: this._context.lineTo(this._x1, this._y1); break;
      }
      if (this._line || (this._line !== 0 && this._point === 1)) this._context.closePath();
      this._line = 1 - this._line;
    },
    //首先移动至起点即第一个点，记录下第一个点和第二个点坐标，连接当前点(第一个点)和((5 * x0 + x1) / 6, (5 * y0 + y1) / 6)，并绘制改点到((x0 + 4 * x1 + x) / 6, (y0 + 4 * y1 + y) / 6)点的三次贝塞尔曲线
    point: function(x, y) {
      x = +x, y = +y;
      switch (this._point) {
        case 0: this._point = 1; this._line ? this._context.lineTo(x, y) : this._context.moveTo(x, y); break;
        case 1: this._point = 2; break;
        case 2: this._point = 3; this._context.lineTo((5 * this._x0 + this._x1) / 6, (5 * this._y0 + this._y1) / 6); // proceed
        default: point(this, x, y); break;
      }
      this._x0 = this._x1, this._x1 = x;
      this._y0 = this._y1, this._y1 = y;
    }
};
// d3.curveBasis
function basis(context) {
    return new Basis(context);
}
```

### d3.curveBasisClosed
通过特定控制点的贝塞尔曲线连接离散的点，并形成一个闭合图形。
```
function BasisClosed(context) {
    this._context = context;
}

BasisClosed.prototype = {
    areaStart: noop,
    areaEnd: noop,
    //(x1, y1)和(x2, y2)用于记录第一个和第二个点的坐标
    lineStart: function() {
      this._x0 = this._x1 = this._x2 = this._x3 = this._x4 =
      this._y0 = this._y1 = this._y2 = this._y3 = this._y4 = NaN;
      this._point = 0;
    },
    lineEnd: function() {
      switch (this._point) {
        //如果只有一个点，则移动到第一个点即该点处并关闭图形
        case 1: {
          this._context.moveTo(this._x2, this._y2);
          this._context.closePath();
          break;
        }
        //如果有两个点，则按如下方式处理
        case 2: {
          this._context.moveTo((this._x2 + 2 * this._x3) / 3, (this._y2 + 2 * this._y3) / 3);
          this._context.lineTo((this._x3 + 2 * this._x2) / 3, (this._y3 + 2 * this._y2) / 3);
          this._context.closePath();
          break;
        }
        //从最后一个点向前绘制，x2,x3,x4分别记录的是前三个点坐标
        case 3: {
          this.point(this._x2, this._y2);
          this.point(this._x3, this._y3);
          this.point(this._x4, this._y4);
          break;
        }
      }
    },
    point: function(x, y) {
      x = +x, y = +y;
      switch (this._point) {
        //记录最初的三个点的坐标
        case 0: this._point = 1; this._x2 = x, this._y2 = y; break;
        case 1: this._point = 2; this._x3 = x, this._y3 = y; break;
        //到最后会绘制一个闭合图形，与初始点连接
        case 2: this._point = 3; this._x4 = x, this._y4 = y; this._context.moveTo((this._x0 + 4 * this._x1 + x) / 6, (this._y0 + 4 * this._y1 + y) / 6); break;
        default: point(this, x, y); break;
      }
      this._x0 = this._x1, this._x1 = x;
      this._y0 = this._y1, this._y1 = y;
    }
  };
//d3.curveBasisClosed
function basisClosed(context) {
    return new BasisClosed(context);
}
```

### d3.curveBasisOpen
```
function BasisOpen(context) {
    this._context = context;
}

BasisOpen.prototype = {
    areaStart: function() {
      this._line = 0;
    },
    areaEnd: function() {
      this._line = NaN;
    },
    lineStart: function() {
      this._x0 = this._x1 =
      this._y0 = this._y1 = NaN;
      this._point = 0;
    },
    lineEnd: function() {
      if (this._line || (this._line !== 0 && this._point === 3)) this._context.closePath();
      this._line = 1 - this._line;
    },
    //记录第一个和第二个点的坐标，从第三个点处开始操作，移动至((x0 + 4 * x1 + x) / 6, (y0 + y1 * 4 + y) / 6) 点处
    point: function(x, y) {
      x = +x, y = +y;
      switch (this._point) {
        case 0: this._point = 1; break;
        case 1: this._point = 2; break;
        case 2: this._point = 3; var x0 = (this._x0 + 4 * this._x1 + x) / 6, y0 = (this._y0 + 4 * this._y1 + y) / 6; this._line ? this._context.lineTo(x0, y0) : this._context.moveTo(x0, y0); break;
        case 3: this._point = 4; // proceed
        default: point(this, x, y); break;
      }
      this._x0 = this._x1, this._x1 = x;
      this._y0 = this._y1, this._y1 = y;
    }
};
//d3.curveBasisOpen
function basisOpen(context) {
    return new BasisOpen(context);
}
```

### d3.curveBundle
根据制定的控制点连接离散的点，用于分层级的关系图中，与`d3.line`一起使用，而不能与`d3.area`使用。
```
Bundle.prototype = {
    lineStart: function() {
      this._x = [];
      this._y = [];
      this._basis.lineStart();
    },
    //结束时开始处理数据并绘制图形
    lineEnd: function() {
      var x = this._x,
          y = this._y,
          j = x.length - 1;

      if (j > 0) {
        var x0 = x[0],
            y0 = y[0],
            //计算起始点到结束点之间x和y的差值
            dx = x[j] - x0,
            dy = y[j] - y0,
            i = -1,
            t;

        while (++i <= j) {
          t = i / j;
          //根据比例确定绘制点的位置，绘制范围在(x0, y0)和(x[j], y[j])之间
          //当x接近0时，结果近似为一条从起始点到结束点的直线；当x接近1时，结果接近d3.curveBasis
          this._basis.point(
            this._beta * x[i] + (1 - this._beta) * (x0 + t * dx),
            this._beta * y[i] + (1 - this._beta) * (y0 + t * dy)
          );
        }
      }

      this._x = this._y = null;
      this._basis.lineEnd();
    },
    //该方法不会绘制图形，只是将数据存入数组在结束绘制时开始处理数据
    point: function(x, y) {
      this._x.push(+x);
      this._y.push(+y);
    }
  };
  //d3.curveBundle
  var bundle = (function custom(beta) {

    function bundle(context) {
      return beta === 1 ? new Basis(context) : new Bundle(context, beta);
    }

    bundle.beta = function(beta) {
      return custom(+beta);
    };

    return bundle;
})(0.85);
```

## Custom Curves
自定义curve函数，需要自定义几个指定的方法。
* curve.areaStart()
    表示一个新的区域的开始，每个区域包含两条线段，topline是数据的顺序绘制，baseline则反向绘制。
* curve.areaEnd()
    表示当前区域的结束。
* curve.lineStart()
    表示一条新的线段的开始，接下来会绘制多个点。
* curve.lineEnd()
    表示当前线段的结束。
* curve.point(x, y)
    在当前线段上根据给定的(x, y)坐标绘制一个新的点。

## Symbols
```
//d3.symbol，默认绘制面积为64的圆形
function symbol() {
    var type = constant$1(circle),
        size = constant$1(64),
        context = null;

    function symbol() {
      var buffer;
      if (!context) context = buffer = path();
      type.apply(this, arguments).draw(context, +size.apply(this, arguments));
      if (buffer) return context = null, buffer + "" || null;
    }
    //设置图形类型
    symbol.type = function(_) {
      return arguments.length ? (type = typeof _ === "function" ? _ : constant$1(_), symbol) : type;
    };
    //设置图形的面积
    symbol.size = function(_) {
      return arguments.length ? (size = typeof _ === "function" ? _ : constant$1(+_), symbol) : size;
    };
    //设置绘制上下文
    symbol.context = function(_) {
      return arguments.length ? (context = _ == null ? null : _, symbol) : context;
    };

    return symbol;
}
```
以内置的`circle`类型为例
```
var circle = {
    //size是该圆形的面积
    draw: function(context, size) {
      var r = Math.sqrt(size / pi$2);
      context.moveTo(r, 0);
      context.arc(0, 0, r, 0, tau$2);
    }
};
```
若自定义type，则应该实现`draw`方法。