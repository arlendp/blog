title: d3.js源码分析——Quadtrees

categories:
- Web开发
- JS
tags:
- Web开发
- D3.js

date: 2016/09/20
folder: web/js
en-title: d3js-source-code-quadtrees
---
四叉树算法用于将二维空间划分成更多的矩形部分，将每个矩形划分成四个大小相等的区域，常用于碰撞检测算法，d3中的forceCollide、forceManyBody等都用到了该数据结构。
<!--more-->

## d3.quadtree
```
/**
   * d3.quadtree用于生成四叉树
   * @param  {object} nodes 将要添加到quadtree中的所有节点
   * @param  {function} x     用于获取节点的x坐标的函数
   * @param  {function} y     用于获取节点的y坐标的函数
   * @return {object}       该quadtree对象
   */
function quadtree(nodes, x, y) {
    var tree = new Quadtree(x == null ? defaultX : x, y == null ? defaultY : y, NaN, NaN, NaN, NaN);
    return nodes == null ? tree : tree.addAll(nodes);
}
  /**
   * 四叉树的构造函数
   * @param {function} x  获取x坐标
   * @param {function} y  获取y坐标
   * @param {number} x0 [x0, y0]到[x1, y1]为该quadtree的矩形区域的范围
   * @param {number} y0 [x0, y0]到[x1, y1]为该quadtree的矩形区域的范围
   * @param {number} x1 [x0, y0]到[x1, y1]为该quadtree的矩形区域的范围
   * @param {number} y1 [x0, y0]到[x1, y1]为该quadtree的矩形区域的范围
   */
function Quadtree(x, y, x0, y0, x1, y1) {
    this._x = x;
    this._y = y;
    this._x0 = x0;
    this._y0 = y0;
    this._x1 = x1;
    this._y1 = y1;
    this._root = undefined;
}
```

## quadtree.add
```
function tree_add(d) {
    var x = +this._x.call(null, d),
        y = +this._y.call(null, d);
    return add(this.cover(x, y), x, y, d);
  }
  /* d3.quadtree的add方法，用于添加node
   * add方法的执行流程如下：
   * 1. 首次添加data0时，由于_root不存在，直接对其赋值data并返回。
   * 2. 第二次添加data1时，node中此时是第一次添加的data0，执行do while循环，为_root赋值一个空数组。判断data1和data0是否是在一个象限(这里将一个区域划分成的四块叫做象限)，如果不在则根据索引分别添加至数组中；
   * 如果在则对该象限再次划分四个区域，继续判断是否在同一个象限。
   * 3. 之后每次添加data时，都会先查找该节点属于哪个象限，根据索引查找node，如果是数组则说明该区域有节点且已经再次划分了象限，则继续进入查找；如果是对象，则说明该区域只有一个节点，此时会对该区域进行划分执行步骤2中的过程；如果是undefined则直接插入该出。
   * 
   */
function add(tree, x, y, d) {
    if (isNaN(x) || isNaN(y)) return tree;

    var parent,
        node = tree._root,
        leaf = {data: d},
        x0 = tree._x0,
        y0 = tree._y0,
        x1 = tree._x1,
        y1 = tree._y1,
        //(xm, ym)表示该区域的中心点
        xm,
        ym,
        xp,
        yp,
        right,
        bottom,
        i,
        j;

    // 如果treenode当前不包含任何node，将leaf作为其节点
    if (!node) return tree._root = leaf, tree;

    // 看(x, y)是否和已有的点在一个象限中，若不是则直接插入，否则往下继续执行
    while (node.length) {
      if (right = x >= (xm = (x0 + x1) / 2)) x0 = xm; else x1 = xm;
      if (bottom = y >= (ym = (y0 + y1) / 2)) y0 = ym; else y1 = ym;
      if (parent = node, !(node = node[i = bottom << 1 | right])) return parent[i] = leaf, tree;
    }

    // 判断(x, y)是否已存在当前节点中
    xp = +tree._x.call(null, node.data);
    yp = +tree._y.call(null, node.data);
    if (x === xp && y === yp) return leaf.next = node, parent ? parent[i] = leaf : tree._root = leaf, tree;

    // 将当前区域进行划分，直至(x, y)和之前的节点不在同一个象限内
    do {
      //parent = parent[i] = new Array(4)会为parent赋值一个空数组，但是由于node = parent，node会形成一个多维数组
      parent = parent ? parent[i] = new Array(4) : tree._root = new Array(4);
      if (right = x >= (xm = (x0 + x1) / 2)) x0 = xm; else x1 = xm;
      if (bottom = y >= (ym = (y0 + y1) / 2)) y0 = ym; else y1 = ym;
    } while ((i = bottom << 1 | right) === (j = (yp >= ym) << 1 | (xp >= xm)));
    return parent[j] = node, parent[i] = leaf, tree;
}
```

## quadtree.addAll
```
//d3.quadtree的addAll方法，先计算数据的范围调整quadtree，之后再添加数据
function addAll(data) {
    var d, i, n = data.length,
        x,
        y,
        xz = new Array(n),
        yz = new Array(n),
        x0 = Infinity,
        y0 = Infinity,
        x1 = -Infinity,
        y1 = -Infinity;

    // 根据_x和_y方法计算data值，得到x、y的范围[x0, x1]和[y0, y1]，即矩形区域的范围
    for (i = 0; i < n; ++i) {
      // this._x和this._y是quadtree中定义的获取x、y坐标的方法
      if (isNaN(x = +this._x.call(null, d = data[i])) || isNaN(y = +this._y.call(null, d))) continue;
      xz[i] = x;
      yz[i] = y;
      if (x < x0) x0 = x;
      if (x > x1) x1 = x;
      if (y < y0) y0 = y;
      if (y > y1) y1 = y;
    }

    // 无效点时的处理
    if (x1 < x0) x0 = this._x0, x1 = this._x1;
    if (y1 < y0) y0 = this._y0, y1 = this._y1;

    // 为quadtree添加范围
    this.cover(x0, y0).cover(x1, y1);

    // 添加node
    for (i = 0; i < n; ++i) {
      add(this, xz[i], yz[i], data[i]);
    }

    return this;
}
```

## quadtree.cover
```
//为quadtree设置区域范围
function tree_cover(x, y) {
    if (isNaN(x = +x) || isNaN(y = +y)) return this;

    var x0 = this._x0,
        y0 = this._y0,
        x1 = this._x1,
        y1 = this._y1;

    // 如果该quadtree范围不存在，则根据当前的(x, y)坐标取范围
    if (isNaN(x0)) {
      x1 = (x0 = Math.floor(x)) + 1;
      y1 = (y0 = Math.floor(y)) + 1;
    }

    // 如果(x, y)在当前范围之外，则扩展当前范围
    else if (x0 > x || x > x1 || y0 > y || y > y1) {
      var z = x1 - x0,
          node = this._root,
          parent,
          i;
      //将该矩形区域的中心看做坐标轴原点，根据x、y坐标轴划分成大小相等的四块区域，0表示右下方，1表示左下方，2表示右上方，3表示左上方。
      //成倍的增长z，扩大当前范围直至(x, y)在当前区域内，在扩大范围的同时不断的构造node数组
      switch (i = (y < (y0 + y1) / 2) << 1 | (x < (x0 + x1) / 2)) {
        case 0: {
          do parent = new Array(4), parent[i] = node, node = parent;
          while (z *= 2, x1 = x0 + z, y1 = y0 + z, x > x1 || y > y1);
          break;
        }
        case 1: {
          do parent = new Array(4), parent[i] = node, node = parent;
          while (z *= 2, x0 = x1 - z, y1 = y0 + z, x0 > x || y > y1);
          break;
        }
        case 2: {
          do parent = new Array(4), parent[i] = node, node = parent;
          while (z *= 2, x1 = x0 + z, y0 = y1 - z, x > x1 || y0 > y);
          break;
        }
        case 3: {
          do parent = new Array(4), parent[i] = node, node = parent;
          while (z *= 2, x0 = x1 - z, y0 = y1 - z, x0 > x || y0 > y);
          break;
        }
      }

      if (this._root && this._root.length) this._root = node;
    }

    // 如果(x, y)已经在当前范围内，则直接返回
    else return this;

    this._x0 = x0;
    this._y0 = y0;
    this._x1 = x1;
    this._y1 = y1;
    return this;
}
```

## quadtree.extend
```
//若有参数且_ = [[x0, y0], [x1, y1]]用于通过cover方法设置quadtree的范围[x0, y0]和[x1, y1]；若没有参数，则以同样的数组形式返回当前区域的范围。
function tree_extent(_) {
    return arguments.length
        ? this.cover(+_[0][0], +_[0][1]).cover(+_[1][0], +_[1][1])
        : isNaN(this._x0) ? undefined : [[this._x0, this._y0], [this._x1, this._y1]];
}
```

## quadtree.data
```
//返回quadtree中所有的node
function tree_data() {
    var data = [];
    this.visit(function(node) {
      if (!node.length) do data.push(node.data); while (node = node.next)
    });
    return data;
}
```

## quadtree.find
```
//查找以(x, y)为中心，radius为半径的范围内离中心最近的点
function tree_find(x, y, radius) {
    var data,
      //(x0, y0)和(x3, y3)表示以(x, y)为中心的矩形搜索区域
        x0 = this._x0,
        y0 = this._y0,
      //(x1, y1)和(x2, y2)表示当前node所在的区域范围
        x1,
        y1,
        x2,
        y2,
        x3 = this._x1,
        y3 = this._y1,
        quads = [],
        node = this._root,
        q,
        i;

    if (node) quads.push(new Quad(node, x0, y0, x3, y3));
    //若没有设置radius则默认为Infinity
    if (radius == null) radius = Infinity;
    else {
      x0 = x - radius, y0 = y - radius;
      x3 = x + radius, y3 = y + radius;
      radius *= radius;
    }

    while (q = quads.pop()) {

      // 如果node不存在或者(x, y)在该node范围外则跳过执行
      if (!(node = q.node)
          //node所在区域与搜索区域不重叠
          //
          //
          || (x1 = q.x0) > x3
          || (y1 = q.y0) > y3
          || (x2 = q.x1) < x0
          || (y2 = q.y1) < y0) continue;

      // 如果node为数组说明已对其进行了区域划分，开始递归查找
      if (node.length) {
        var xm = (x1 + x2) / 2,
            ym = (y1 + y2) / 2;
        // node数组中0表示区域左上角，1表示右上角，2表示左下角，3表示右下角
        quads.push(
          new Quad(node[3], xm, ym, x2, y2),
          new Quad(node[2], x1, ym, xm, y2),
          new Quad(node[1], xm, y1, x2, ym),
          new Quad(node[0], x1, y1, xm, ym)
        );

        // 判断(x, y)所在的象限，并将该象限对应的node与栈顶的数据交换位置，如果是左上角区域及表示已是栈顶则不用处理
        if (i = (y >= ym) << 1 | (x >= xm)) {
          q = quads[quads.length - 1];
          quads[quads.length - 1] = quads[quads.length - 1 - i];
          quads[quads.length - 1 - i] = q;
        }
      }

      // 当查找到在搜索范围内的点后，缩小搜索范围
      else {
        var dx = x - +this._x.call(null, node.data),
            dy = y - +this._y.call(null, node.data),
            d2 = dx * dx + dy * dy;
        if (d2 < radius) {
          var d = Math.sqrt(radius = d2);
          x0 = x - d, y0 = y - d;
          x3 = x + d, y3 = y + d;
          data = node.data;
        }
      }
    }

    return data;
}
```

## quadtree.remove
```
function tree_remove(d) {
    if (isNaN(x = +this._x.call(null, d)) || isNaN(y = +this._y.call(null, d))) return this; 

    var parent,
        node = this._root,
        retainer,
        previous,
        next,
        x0 = this._x0,
        y0 = this._y0,
        x1 = this._x1,
        y1 = this._y1,
        x,
        y,
        xm,
        ym,
        right,
        bottom,
        i,
        j;

    if (!node) return this;

    // 当node中有多个点时，进入查找
    if (node.length) while (true) {
      //计算(x, y)所在的象限
      if (right = x >= (xm = (x0 + x1) / 2)) x0 = xm; else x1 = xm;
      if (bottom = y >= (ym = (y0 + y1) / 2)) y0 = ym; else y1 = ym;
      //如果对应的象限中没有点，则说明查找不到该点，直接返回。
      if (!(parent = node, node = node[i = bottom << 1 | right])) return this;
      //如果该象限只有一个点则跳出循环往下执行。
      if (!node.length) break;
      //
      if (parent[(i + 1) & 3] || parent[(i + 2) & 3] || parent[(i + 3) & 3]) retainer = parent, j = i;
    }

    // TODO: 这里存在一个问题，由于node.data和d都为数组，但是两个数据是不同的指针，因此这里不会相等
    while (node.data !== d) if (!(previous = node, node = node.next)) return this;
    if (next = node.next) delete node.next;

    // If there are multiple coincident points, remove just the point.
    if (previous) return (next ? previous.next = next : delete previous.next), this;

    // If this is the root point, remove it.
    if (!parent) return this._root = next, this;

    // Remove this leaf.
    next ? parent[i] = next : delete parent[i];

    // If the parent now contains exactly one leaf, collapse superfluous parents.
    if ((node = parent[0] || parent[1] || parent[2] || parent[3])
        && node === (parent[3] || parent[2] || parent[1] || parent[0])
        && !node.length) {
      if (retainer) retainer[j] = node;
      else this._root = node;
    }

    return this;
}
```

## quadtree.visit
```
//采用先序遍历的方式，如果callback返回true，则执行不再访问其子节点；否则继续访问其子节点
function tree_visit(callback) {
    var quads = [], q, node = this._root, child, x0, y0, x1, y1;
    if (node) quads.push(new Quad(node, this._x0, this._y0, this._x1, this._y1));
    while (q = quads.pop()) {
      if (!callback(node = q.node, x0 = q.x0, y0 = q.y0, x1 = q.x1, y1 = q.y1) && node.length) {
        var xm = (x0 + x1) / 2, ym = (y0 + y1) / 2;
        if (child = node[3]) quads.push(new Quad(child, xm, ym, x1, y1));
        if (child = node[2]) quads.push(new Quad(child, x0, ym, xm, y1));
        if (child = node[1]) quads.push(new Quad(child, xm, y0, x1, ym));
        if (child = node[0]) quads.push(new Quad(child, x0, y0, xm, ym));
      }
    }
    return this;
}
```

## quadtree.visitAfter
```
//采用后序遍历的方式，先将所有节点存入数组中，然后依次对所有节点进行操作。
function tree_visitAfter(callback) {
    var quads = [], next = [], q;
    if (this._root) quads.push(new Quad(this._root, this._x0, this._y0, this._x1, this._y1));
    while (q = quads.pop()) {
      var node = q.node;
      if (node.length) {
        var child, x0 = q.x0, y0 = q.y0, x1 = q.x1, y1 = q.y1, xm = (x0 + x1) / 2, ym = (y0 + y1) / 2;
        if (child = node[0]) quads.push(new Quad(child, x0, y0, xm, ym));
        if (child = node[1]) quads.push(new Quad(child, xm, y0, x1, ym));
        if (child = node[2]) quads.push(new Quad(child, x0, ym, xm, y1));
        if (child = node[3]) quads.push(new Quad(child, xm, ym, x1, y1));
      }
      next.push(q);
    }
    while (q = next.pop()) {
      callback(q.node, q.x0, q.y0, q.x1, q.y1);
    }
    return this;
}
```

## quadtree.copy
```
//对quadtree进行复制，但是是通过引用来复制的而不是复制值。
treeProto.copy = function() {
    var copy = new Quadtree(this._x, this._y, this._x0, this._y0, this._x1, this._y1),
        node = this._root,
        nodes,
        child;

    if (!node) return copy;

    if (!node.length) return copy._root = leaf_copy(node), copy;

    nodes = [{source: node, target: copy._root = new Array(4)}];
    while (node = nodes.pop()) {
      for (var i = 0; i < 4; ++i) {
        if (child = node.source[i]) {
          if (child.length) nodes.push({source: child, target: node.target[i] = new Array(4)});
          else node.target[i] = leaf_copy(child);
        }
      }
    }

    return copy;
};

//复制叶子节点
function leaf_copy(leaf) {
    var copy = {data: leaf.data}, next = copy;
    while (leaf = leaf.next) next = next.next = {data: leaf.data};
    return copy;
}
```

## quad对象
在quadtree的一些方法中使用到了quad对象用于存储quadtree中的node信息，包括node的值和其所在区域的坐标范围。
```
/**
   * Quad构造函数
   * @param {object} node 节点数据
   * @param {number} x0   该节点的区域坐标范围
   * @param {number} y0   该节点的区域坐标范围
   * @param {number} x1   该节点的区域坐标范围
   * @param {number} y1   该节点的区域坐标范围
   *
   * quad对象在quadtree的node中的位置如下：
   * 
   *        |
   *    0   |    1
   *        |
   * -------|--------
   *        |
   *    2   |    3
   *        |
   */
function Quad(node, x0, y0, x1, y1) {
    this.node = node;
    this.x0 = x0;
    this.y0 = y0;
    this.x1 = x1;
    this.y1 = y1;
}
```



