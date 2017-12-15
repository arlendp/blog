title: d3.js源码分析——Chord

categories:
- Web开发
- JS
tags:
- Web开发
- D3.js

date: 2016/09/01
folder: web/js
en-title: d3js-source-code-chord
---
d3的chord部分用于将关系或网络流绘制成一种圆形布局。
<!--more-->

![chord图](https://raw.githubusercontent.com/d3/d3-chord/master/img/chord.png)

这部分内容分为两个方面，一方面是构造一个弦布局，另一方面是构造一个产生带状图形的生成器。

## chord
chord的源码如下：
```
function chord() {
    var padAngle = 0,
        sortGroups = null,
        sortSubgroups = null,
        sortChords = null;

    function chord(matrix) {
      var n = matrix.length,
          //matrix中每组数的总和
          groupSums = [],
          groupIndex = range(n),
          subgroupIndex = [],
          chords = [],
          groups = chords.groups = new Array(n),
          subgroups = new Array(n * n),
          k,
          x,
          x0,
          dx,
          i,
          j;

      // 计算每组数的和以及所有数值的总和
      k = 0, i = -1; while (++i < n) {
        x = 0, j = -1; while (++j < n) {
          x += matrix[i][j];
        }
        groupSums.push(x);
        subgroupIndex.push(range(n));
        k += x;
      }

      // sortGroups函数根据每组数据和的大小对groupIndex进行排序
      if (sortGroups) groupIndex.sort(function(a, b) {
        return sortGroups(groupSums[a], groupSums[b]);
      });

      // sortSubgroups函数根据每个数据大小在该组内进行索引的排序
      if (sortSubgroups) subgroupIndex.forEach(function(d, i) {
        d.sort(function(a, b) {
          return sortSubgroups(matrix[i][a], matrix[i][b]);
        });
      });

      // 计算除去padAngle之后的单位弧度（每单位数值对应的弧度）
      k = max$1(0, tau$3 - padAngle * n) / k;
      dx = k ? padAngle : tau$3 / n;

      // 计算每个数据对应的startAngle和endAngle
      x = 0, i = -1; while (++i < n) {
        x0 = x, j = -1; while (++j < n) {
          var di = groupIndex[i],
              dj = subgroupIndex[di][j],
              v = matrix[di][dj],
              // startAngle
              a0 = x,
              // 计算endAngle
              a1 = x += v * k;
          //记录matrix中每个数据在弦图中的信息
          subgroups[dj * n + di] = {
            index: di,
            subindex: dj,
            startAngle: a0,
            endAngle: a1,
            value: v
          };
        }
        //记录matrix中每组数据在弦图中的信息
        groups[di] = {
          index: di,
          startAngle: x0,
          endAngle: x,
          value: groupSums[di]
        };
        //考虑弦图中每组之间的间距
        x += dx;
      }

      // 产生source和target
      i = -1; while (++i < n) {
        j = i - 1; while (++j < n) {
          var source = subgroups[j * n + i],
              target = subgroups[i * n + j];
          if (source.value || target.value) {
            //将value大的设置为source，小的设置为target
            chords.push(source.value < target.value
                ? {source: target, target: source}
                : {source: source, target: target});
          }
        }
      }

      return sortChords ? chords.sort(sortChords) : chords;
    }
    // 设置相邻组之间的间距，以弧度形式表示
    chord.padAngle = function(_) {
      return arguments.length ? (padAngle = max$1(0, _), chord) : padAngle;
    };
    // 对groupIndex进行排序
    chord.sortGroups = function(_) {
      return arguments.length ? (sortGroups = _, chord) : sortGroups;
    };
    // 对subgroupIndex进行排序
    chord.sortSubgroups = function(_) {
      return arguments.length ? (sortSubgroups = _, chord) : sortSubgroups;
    };
    //对chords数组进行排序，影响的是chord的层叠顺序，两根弦重叠，重叠部分后面的会覆盖掉前面的
    chord.sortChords = function(_) {
      return arguments.length ? (_ == null ? sortChords = null : (sortChords = compareValue(_))._ = _, chord) : sortChords && sortChords._;
    };

    return chord;
}
```
chord函数最终会得到一个包含多组`source`和`target`对象的数组以及`groups`数组，通过将该结果传递给`d3.arc`来绘制弦图外层的圆弧，而其内部的带状图则通过`d3.ribbon`来实现。

## ribbon
用于绘制弦图中间部分表示各块之间联系的带状区域。
```
  function ribbon() {
    var source = defaultSource,
        target = defaultTarget,
        radius = defaultRadius$1,
        startAngle = defaultStartAngle,
        endAngle = defaultEndAngle,
        context = null;

    function ribbon() {
      var buffer,
          argv = slice$5.call(arguments),
          //source对象
          s = source.apply(this, argv),
          //target对象
          t = target.apply(this, argv),
          //带状图形中弧线的半径
          sr = +radius.apply(this, (argv[0] = s, argv)),

          sa0 = startAngle.apply(this, argv) - halfPi$2,
          sa1 = endAngle.apply(this, argv) - halfPi$2,
          sx0 = sr * cos(sa0),
          sy0 = sr * sin(sa0),
          tr = +radius.apply(this, (argv[0] = t, argv)),
          ta0 = startAngle.apply(this, argv) - halfPi$2,
          ta1 = endAngle.apply(this, argv) - halfPi$2;
      //构造path对象，用于存储路径
      if (!context) context = buffer = path();
      //移动到startAngle对应的起始点
      context.moveTo(sx0, sy0);
      //向endAngle位置画弧线
      context.arc(0, 0, sr, sa0, sa1);
      //判断source和target是否是同个位置
      if (sa0 !== ta0 || sa1 !== ta1) { // TODO sr !== tr?
        // 从source的endAngle位置绘制贝塞尔曲线至target的startAngle处
        context.quadraticCurveTo(0, 0, tr * cos(ta0), tr * sin(ta0));
        //target的startAngle绘制圆弧至endAngle位置
        context.arc(0, 0, tr, ta0, ta1);
      }
      //以(0, 0)为控制点绘制贝塞尔曲线至startAngle位置
      context.quadraticCurveTo(0, 0, sx0, sy0);
      context.closePath();

      if (buffer) return context = null, buffer + "" || null;
    }

    ribbon.radius = function(_) {
      return arguments.length ? (radius = typeof _ === "function" ? _ : constant$11(+_), ribbon) : radius;
    };

    ribbon.startAngle = function(_) {
      return arguments.length ? (startAngle = typeof _ === "function" ? _ : constant$11(+_), ribbon) : startAngle;
    };

    ribbon.endAngle = function(_) {
      return arguments.length ? (endAngle = typeof _ === "function" ? _ : constant$11(+_), ribbon) : endAngle;
    };

    ribbon.source = function(_) {
      return arguments.length ? (source = _, ribbon) : source;
    };

    ribbon.target = function(_) {
      return arguments.length ? (target = _, ribbon) : target;
    };
    //设置当前路径上下文
    ribbon.context = function(_) {
      return arguments.length ? ((context = _ == null ? null : _), ribbon) : context;
    };

    return ribbon;
}
```



