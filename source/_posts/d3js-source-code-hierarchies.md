title: d3.js源码分析——Hierarchies

categories:
- Web开发
- JS
tags:
- Web开发
- D3.js

date: 2016/09/26
folder: web/js
en-title: d3js-source-code-hierarchies
---
d3的hierarchy模块用于层级图的计算，会将输入的数据计算并转换成指定的层级格式提供给开发者使用。为了表示这种数据间的层级关系，该模块在内部使用了四叉树这种数据结构。
<!--more-->

## Hierarchy
用于计算层级数据，在层级图中使用。
### d3.hierarchy
```
/**
   * 处理层级数据
   * @param  {object} data     输入的数据
   * @param  {function} children 用于获取data中的children数据的函数
   * @return {object}          处理后的层级数据
   */
function hierarchy(data, children) {
    var root = new Node(data),
        valued = +data.value && (root.value = data.value),
        node,
        nodes = [root],
        child,
        childs,
        i,
        n;

    if (children == null) children = defaultChildren;
    //先处理父节点，后处理子节点，构造成node对象
    while (node = nodes.pop()) {
      if (valued) node.value = +node.data.value;
      if ((childs = children(node.data)) && (n = childs.length)) {
        node.children = new Array(n);
        for (i = n - 1; i >= 0; --i) {
          nodes.push(child = node.children[i] = new Node(childs[i]));
          child.parent = node;
          child.depth = node.depth + 1;
        }
      }
    }

    return root.eachBefore(computeHeight);
}
```
node用于表示hierarchy中的节点对象。
```
/* node构造函数
   * data: 该node相关联的数据
   * depth: 该节点所在的层级数，根节点为0，子节点逐渐递增
   * height: 该节点与其最远的子节点之间的距离，叶子节点为0
   * parent: 该节点的父节点
   */
function Node(data) {
    this.data = data;
    this.depth =
    this.height = 0;
    this.parent = null;
}
```
node的原型方法:
```
//返回当前节点的所有父级节点，以当前节点开始逐级向上查找直至根节点
  function node_ancestors() {
    var node = this, nodes = [node];
    while (node = node.parent) {
      nodes.push(node);
    }
    return nodes;
  }
  //返回当前节点的所有子节点，包括当前节点
  function node_descendants() {
    var nodes = [];
    this.each(function(node) {
      nodes.push(node);
    });
    return nodes;
  }
  //返回当前节点包含的所有叶子节点
  function node_leaves() {
    var leaves = [];
    this.eachBefore(function(node) {
      if (!node.children) {
        leaves.push(node);
      }
    });
    return leaves;
  }
  //以节点的父节点和本身构成一个新的对象，返回包含所有该种对象的数组
  function node_links() {
    var root = this, links = [];
    root.each(function(node) {
      if (node !== root) { // Don’t include the root’s parent, if any.
        links.push({source: node.parent, target: node});
      }
    });
    return links;
  }
```
根据node的不同的遍历方式而有不同的方法
```
//广度优先？？？
  function node_each(callback) {
    var node = this, current, next = [node], children, i, n;
    do {
      current = next.reverse(), next = [];
      while (node = current.pop()) {
        callback(node), children = node.children;
        if (children) for (i = 0, n = children.length; i < n; ++i) {
          next.push(children[i]);
        }
      }
    } while (next.length);
    return this;
  }
  // 先处理回调函数，后访问子节点
  function node_eachBefore(callback) {
    var node = this, nodes = [node], children, i;
    while (node = nodes.pop()) {
      callback(node), children = node.children;
      if (children) for (i = children.length - 1; i >= 0; --i) {
        nodes.push(children[i]);
      }
    }
    return this;
  }
  //先访问所有节点，之后逐个执行回调
  function node_eachAfter(callback) {
    var node = this, nodes = [node], next = [], children, i, n;
    while (node = nodes.pop()) {
      next.push(node), children = node.children;
      if (children) for (i = 0, n = children.length; i < n; ++i) {
        nodes.push(children[i]);
      }
    }
    while (node = next.pop()) {
      callback(node);
    }
    return this;
  }
```
而在这些遍历方法的基础上构造出来的方法
```
//通过value函数对每个节点的data进行计算，节点的value值为其自己的value值加上所有子节点的value值之和。
  function node_sum(value) {
    return this.eachAfter(function(node) {
      var sum = +value(node.data) || 0,
          children = node.children,
          i = children && children.length;
      while (--i >= 0) sum += children[i].value;
      node.value = sum;
    });
  }
  //对所有节点的子节点进行排序，内部调用Array的原型链方法
  function node_sort(compare) {
    return this.eachBefore(function(node) {
      if (node.children) {
        node.children.sort(compare);
      }
    });
  }
  //计算当前node到end的最短路径，返回的数组从当前节点的父节点开始到公共节点，然后到目标节点
  function node_path(end) {
    var start = this,
        ancestor = leastCommonAncestor(start, end),
        nodes = [start];
    //从start开始向上查找至ancestor
    while (start !== ancestor) {
      start = start.parent;
      nodes.push(start);
    }
    var k = nodes.length;
    while (end !== ancestor) {

      nodes.splice(k, 0, end);
      end = end.parent;
    }
    return nodes;
  }
  //返回最近的相同的祖先节点
  function leastCommonAncestor(a, b) {
    if (a === b) return a;
    var aNodes = a.ancestors(),
        bNodes = b.ancestors(),
        c = null;
    a = aNodes.pop();
    b = bNodes.pop();
    while (a === b) {
      c = a;
      a = aNodes.pop();
      b = bNodes.pop();
    }
    return c;
  }
  //复制一份相同的node
  function node_copy() {
    return hierarchy(this).eachBefore(copyData);
  }
  function copyData(node) {
    node.data = node.data.data;
  }
```
## Stratify
将数据转化为层级形式。若数据格式已经是如下形式:
```
var data = {
        "name": "中国",
        "children": [{
                "name": "浙江",
                "children": [{
                    "name": "杭州"
                }, {
                    "name": "宁波"
                }, {
                    "name": "温州"
                }, {
                    "name": "绍兴"
                }]
            },

            {
                "name": "广西",
                "children": [{
                    "name": "桂林"
                }, {
                    "name": "南宁"
                }, {
                    "name": "柳州"
                }, {
                    "name": "防城港"
                }]
            }]
    };
```
则可以直接传入上述`d3.hierarchy`方法来构造层级数据。若不是则用`d3.stratify`方法来处理，其关键部分是'id'和'parentId'方法。
```
//d3.stratify
function stratify() {
    var id = defaultId,
        parentId = defaultParentId;

    function stratify(data) {
      var d,
          i,
          n = data.length,
          root,
          parent,
          node,
          nodes = new Array(n),
          nodeId,
          nodeKey,
          nodeByKey = {};

      for (i = 0; i < n; ++i) {
        //将data中每个数据构造成node对象
        d = data[i], node = nodes[i] = new Node(d);
        //根据id函数获取data的id
        if ((nodeId = id(d, i, data)) != null && (nodeId += "")) {
          nodeKey = keyPrefix$1 + (node.id = nodeId);
          nodeByKey[nodeKey] = nodeKey in nodeByKey ? ambiguous : node;
        }
      }

      for (i = 0; i < n; ++i) {
        node = nodes[i], nodeId = parentId(data[i], i, data);
        //parentId为空时认为该node为根节点，但只能存在一个parentId为空的节点
        if (nodeId == null || !(nodeId += "")) {
          if (root) throw new Error("multiple roots");
          root = node;
        } else {
          parent = nodeByKey[keyPrefix$1 + nodeId];
          //如果记录中没有parentId，则抛出异常
          if (!parent) throw new Error("missing: " + nodeId);
          if (parent === ambiguous) throw new Error("ambiguous: " + nodeId);
          //将该节点添加至parentId对应节点的children属性中
          if (parent.children) parent.children.push(node);
          else parent.children = [node];
          //为node节点添加parent属性
          node.parent = parent;
        }
      }

      if (!root) throw new Error("no root");
      root.parent = preroot;
      //计算节点的depth和height值
      root.eachBefore(function(node) { node.depth = node.parent.depth + 1; --n; }).eachBefore(computeHeight);
      root.parent = null;
      if (n > 0) throw new Error("cycle");

      return root;
    }
    //设置获取id的函数
    stratify.id = function(x) {
      return arguments.length ? (id = required(x), stratify) : id;
    };
    //设置获取父节点id的函数
    stratify.parentId = function(x) {
      return arguments.length ? (parentId = required(x), stratify) : parentId;
    };

    return stratify;
}
```

## Cluster
用于绘制集群图，它会将所有的叶子节点放置在相同的深度，即所有的叶子节点会对齐。这里返回的结果包含(x, y)坐标，即可以直接用于绘制。
```
//d3.cluster
function cluster() {
    var separation = defaultSeparation,
        dx = 1,
        dy = 1,
        nodeSize = false;

    function cluster(root) {
      var previousNode,
          x = 0;

      root.eachAfter(function(node) {
        var children = node.children;
        if (children) {
          //计算非叶子节点的x、y坐标，与其子节点相关
          node.x = meanX(children);
          node.y = maxY(children);
        } else {
          // 如果是叶子节点，则其y坐标为0，x坐标则根据当前节点与前一个节点是否含有相同的父节点来设置
          node.x = previousNode ? x += separation(node, previousNode) : 0;
          node.y = 0;
          previousNode = node;
        }
      });

      var left = leafLeft(root),
          right = leafRight(root),
          x0 = left.x - separation(left, right) / 2,
          x1 = right.x + separation(right, left) / 2;

      // 根据size大小对节点的x, y坐标进行调整
      return root.eachAfter(nodeSize ? function(node) {
        //nodeSize为true时，将root放置于(0, 0)位置
        node.x = (node.x - root.x) * dx;
        node.y = (root.y - node.y) * dy;
      } : function(node) {
        //否则，按比例对节点坐标进行调整
        node.x = (node.x - x0) / (x1 - x0) * dx;
        node.y = (1 - (root.y ? node.y / root.y : 1)) * dy;
      });
    }
    //separation用于将相邻的叶子节点进行分离
    cluster.separation = function(x) {
      return arguments.length ? (separation = x, cluster) : separation;
    };
    //以数组的形式设置cluster的范围大小
    cluster.size = function(x) {
      return arguments.length ? (nodeSize = false, dx = +x[0], dy = +x[1], cluster) : (nodeSize ? null : [dx, dy]);
    };

    cluster.nodeSize = function(x) {
      return arguments.length ? (nodeSize = true, dx = +x[0], dy = +x[1], cluster) : (nodeSize ? [dx, dy] : null);
    };

    return cluster;
}
```

## Tree
用于产生树状布局，以根节点位置为基准，逐级对齐。
```

```

## Treemap
用于产生矩形式树状结构图。
```
//d3.treemap
function index$1() {
    var tile = squarify,
        round = false,
        dx = 1,
        dy = 1,
        paddingStack = [0],
        paddingInner = constantZero,
        paddingTop = constantZero,
        paddingRight = constantZero,
        paddingBottom = constantZero,
        paddingLeft = constantZero;

    function treemap(root) {
      root.x0 =
      root.y0 = 0;
      root.x1 = dx;
      root.y1 = dy;
      root.eachBefore(positionNode);
      paddingStack = [0];
      if (round) root.eachBefore(roundNode);
      return root;
    }

    function positionNode(node) {
      var p = paddingStack[node.depth],
          //将有效区域根据padding来缩小
          x0 = node.x0 + p,
          y0 = node.y0 + p,
          x1 = node.x1 - p,
          y1 = node.y1 - p;
      //处理padding过大的情况
      if (x1 < x0) x0 = x1 = (x0 + x1) / 2;
      if (y1 < y0) y0 = y1 = (y0 + y1) / 2;
      node.x0 = x0;
      node.y0 = y0;
      node.x1 = x1;
      node.y1 = y1;
      if (node.children) {
        p = paddingStack[node.depth + 1] = paddingInner(node) / 2;
        //在调整了范围的基础上根据特定的padding再次进行调整
        x0 += paddingLeft(node) - p;
        y0 += paddingTop(node) - p;
        x1 -= paddingRight(node) - p;
        y1 -= paddingBottom(node) - p;
        if (x1 < x0) x0 = x1 = (x0 + x1) / 2;
        if (y1 < y0) y0 = y1 = (y0 + y1) / 2;
        tile(node, x0, y0, x1, y1);
      }
    }

    treemap.round = function(x) {
      return arguments.length ? (round = !!x, treemap) : round;
    };

    treemap.size = function(x) {
      return arguments.length ? (dx = +x[0], dy = +x[1], treemap) : [dx, dy];
    };
    //设置tile函数，默认是d3.treemapSquarify，即按黄金分割比进行划分
    treemap.tile = function(x) {
      return arguments.length ? (tile = required(x), treemap) : tile;
    };
    //同时设置paddingInner和paddingOuter
    treemap.padding = function(x) {
      return arguments.length ? treemap.paddingInner(x).paddingOuter(x) : treemap.paddingInner();
    };

    treemap.paddingInner = function(x) {
      return arguments.length ? (paddingInner = typeof x === "function" ? x : constant$5(+x), treemap) : paddingInner;
    };
    //paddingOuter是由四个方向的padding构成
    treemap.paddingOuter = function(x) {
      return arguments.length ? treemap.paddingTop(x).paddingRight(x).paddingBottom(x).paddingLeft(x) : treemap.paddingTop();
    };

    treemap.paddingTop = function(x) {
      return arguments.length ? (paddingTop = typeof x === "function" ? x : constant$5(+x), treemap) : paddingTop;
    };

    treemap.paddingRight = function(x) {
      return arguments.length ? (paddingRight = typeof x === "function" ? x : constant$5(+x), treemap) : paddingRight;
    };

    treemap.paddingBottom = function(x) {
      return arguments.length ? (paddingBottom = typeof x === "function" ? x : constant$5(+x), treemap) : paddingBottom;
    };

    treemap.paddingLeft = function(x) {
      return arguments.length ? (paddingLeft = typeof x === "function" ? x : constant$5(+x), treemap) : paddingLeft;
    };

    return treemap;
}
```

`d3.treemap`中会对node进行区块划分，其中主要是用到`tile`函数来实现模块划分的逻辑，默认是`d3.treemapSquarify`即按黄金分割比进行区块的分割。
### d3.treemapSquarify
通过黄金分割比来对treemap进行分割。
```
//黄金分割比
var phi = (1 + Math.sqrt(5)) / 2;
//按黄金分割比对treenode区块进行划分
var squarify = (function custom(ratio) {

    function squarify(parent, x0, y0, x1, y1) {
      squarifyRatio(ratio, parent, x0, y0, x1, y1);
    }

    squarify.ratio = function(x) {
      return custom((x = +x) > 1 ? x : 1);
    };

    return squarify;
})(phi);

function squarifyRatio(ratio, parent, x0, y0, x1, y1) {
    var rows = [],
        nodes = parent.children,
        row,
        nodeValue,
        i0 = 0,
        i1,
        n = nodes.length,
        dx, dy,
        value = parent.value,
        sumValue,
        minValue,
        maxValue,
        newRatio,
        minRatio,
        alpha,
        beta;

    while (i0 < n) {
      dx = x1 - x0, dy = y1 - y0;
      minValue = maxValue = sumValue = nodes[i0].value;
      //根据长宽比和黄金分割比计算alpha，其值不受子节点影响
      alpha = Math.max(dy / dx, dx / dy) / (value * ratio);
      beta = sumValue * sumValue * alpha;
      minRatio = Math.max(maxValue / beta, beta / minValue);

      // 当ratio不增加时，添加node
      for (i1 = i0 + 1; i1 < n; ++i1) {
        sumValue += nodeValue = nodes[i1].value;
        if (nodeValue < minValue) minValue = nodeValue;
        if (nodeValue > maxValue) maxValue = nodeValue;
        beta = sumValue * sumValue * alpha;
        newRatio = Math.max(maxValue / beta, beta / minValue);
        //不会添加使ratio增加的node，如果不满足退出循环
        if (newRatio > minRatio) { sumValue -= nodeValue; break; }
        minRatio = newRatio;
      }

      // 调整node的范围并确定分割的方向
      rows.push(row = {value: sumValue, dice: dx < dy, children: nodes.slice(i0, i1)});
      //当node区域长比宽短时
      if (row.dice) treemapDice(row, x0, y0, x1, value ? y0 += dy * sumValue / value : y1);
      //当node区域宽比长短时
      else treemapSlice(row, x0, y0, value ? x0 += dx * sumValue / value : x1, y1);
      //value等于剩余未分配位置的node的value之和，i0也从未分配的node开始
      value -= sumValue, i0 = i1;
    }

    return rows;
}
```

### d3.treemapBinary
将treemap区域根据value值大致进行二等分，使两边的value值之和尽量相近。
```
//d3.treemapBinary
function binary(parent, x0, y0, x1, y1) {
    var nodes = parent.children,
        i, n = nodes.length,
        sum, sums = new Array(n + 1);

    for (sums[0] = sum = i = 0; i < n; ++i) {
      sums[i + 1] = sum += nodes[i].value;
    }

    partition(0, n, parent.value, x0, y0, x1, y1);

    function partition(i, j, value, x0, y0, x1, y1) {
      if (i >= j - 1) {
        var node = nodes[i];
        node.x0 = x0, node.y0 = y0;
        node.x1 = x1, node.y1 = y1;
        return;
      }

      var valueOffset = sums[i],
          //加上前面的已经计算过的value值作为偏移量，这样才能将sums[mid]跟valueTarget进行比较
          valueTarget = (value / 2) + valueOffset,
          k = i + 1,
          hi = j - 1;
      //二分法查找
      while (k < hi) {
        var mid = k + hi >>> 1;
        if (sums[mid] < valueTarget) k = mid + 1;
        else hi = mid;
      }

      var valueLeft = sums[k] - valueOffset,
          valueRight = value - valueLeft;
      //当矩形较高时，进行上下分割
      if ((y1 - y0) > (x1 - x0)) {
        //根据左右的value和进行坐标划分
        var yk = (y0 * valueRight + y1 * valueLeft) / value;
        partition(i, k, valueLeft, x0, y0, x1, yk);
        partition(k, j, valueRight, x0, yk, x1, y1);
      } 
      //否则进行左右分割
      else {
        var xk = (x0 * valueRight + x1 * valueLeft) / value;
        partition(i, k, valueLeft, x0, y0, xk, y1);
        partition(k, j, valueRight, xk, y0, x1, y1);
      }
    }
}
```

## Partition
节点以矩形区域的形式展现，节点间的相对位置可以看出其层级关系。同时区块的大小可以反映value值大小。
```
//d3.partition
function partition() {
    var dx = 1,
        dy = 1,
        padding = 0,
        round = false;

    function partition(root) {
      var n = root.height + 1;
      root.x0 =
      root.y0 = padding;
      root.x1 = dx;
      root.y1 = dy / n;
      root.eachBefore(positionNode(dy, n));
      if (round) root.eachBefore(roundNode);
      return root;
    }

    function positionNode(dy, n) {
      return function(node) {
        if (node.children) {
          treemapDice(node, node.x0, dy * (node.depth + 1) / n, node.x1, dy * (node.depth + 2) / n);
        }
        var x0 = node.x0,
            y0 = node.y0,
            //这里减去padding用于与下个兄弟节点分开
            x1 = node.x1 - padding,
            y1 = node.y1 - padding;
        if (x1 < x0) x0 = x1 = (x0 + x1) / 2;
        if (y1 < y0) y0 = y1 = (y0 + y1) / 2;
        node.x0 = x0;
        node.y0 = y0;
        node.x1 = x1;
        node.y1 = y1;
      };
    }

    partition.round = function(x) {
      return arguments.length ? (round = !!x, partition) : round;
    };

    partition.size = function(x) {
      return arguments.length ? (dx = +x[0], dy = +x[1], partition) : [dx, dy];
    };
    //padding用于将节点的相邻子节点分开
    partition.padding = function(x) {
      return arguments.length ? (padding = +x, partition) : padding;
    };

    return partition;
}
```

## Pack
通过多个封闭圆来表现层级图。
