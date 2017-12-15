title: d3.js源码分析——Force

categories:
- Web开发
- JS
tags:
- Web开发
- D3.js

date: 2016/09/18
folder: web/js
en-title: d3js-source-code-force
---
该模块用于模拟在粒子上的物理作用力，常用于网络图和层级图。
<!--more-->

## Simulation

### d3.forceSimulation
根据指定的nodes值创建一个新的simulation，此时还没有设置force函数。
```
//d3.forceSimulation用于设置节点和相关参数
function simulation(nodes) {
    var simulation,
        //alpha表示simulation当前的状态
        alpha = 1,
        alphaMin = 0.001,
        //alphaDecay表示alpha每次的衰减率
        alphaDecay = 1 - Math.pow(alphaMin, 1 / 300),
        //alphaTarget表示最终要稳定时的状态
        alphaTarget = 0,
        //velocityDecay表示速度的衰退率
        velocityDecay = 0.6,
        //用于存储force函数
        forces = map$1(),
        stepper = timer(step),
        //simulation包含以下两种类型的事件
        event = dispatch("tick", "end");

    if (nodes == null) nodes = [];

    function step() {
      tick();
      //自定义的tick函数，在这里被调用
      event.call("tick", simulation);
      //当alpha小于临界值即alphaMin时，停止计时
      if (alpha < alphaMin) {
        stepper.stop();
        //自定义的end函数在这里调用
        event.call("end", simulation);
      }
    }

    function tick() {
      var i, n = nodes.length, node;

      alpha += (alphaTarget - alpha) * alphaDecay;
      //alpha用于force中对速度vx和vy进行设置
      forces.each(function(force) {
        force(alpha);
      });

      for (i = 0; i < n; ++i) {
        node = nodes[i];
        //fx和fy是node的固定点，如果设置了该属性则node会固定在该位置
        //这里简化了物理作用力，将当前位置坐标加上当前速度得到下一步的位置坐标
        if (node.fx == null) node.x += node.vx *= velocityDecay;
        else node.x = node.fx, node.vx = 0;
        if (node.fy == null) node.y += node.vy *= velocityDecay;
        else node.y = node.fy, node.vy = 0;
      }
    }
    //对nodes进行处理
    function initializeNodes() {
      for (var i = 0, n = nodes.length, node; i < n; ++i) {
        node = nodes[i], node.index = i;
        //如果node中不含x、 y值，则按默认方法计算。
        if (isNaN(node.x) || isNaN(node.y)) {
          var radius = initialRadius * Math.sqrt(i), angle = i * initialAngle;
          node.x = radius * Math.cos(angle);
          node.y = radius * Math.sin(angle);
        }
        //如果不含vx、vy值，则默认为0。
        if (isNaN(node.vx) || isNaN(node.vy)) {
          node.vx = node.vy = 0;
        }
      }
    }

    function initializeForce(force) {
      if (force.initialize) force.initialize(nodes);
      return force;
    }

    initializeNodes();

    return simulation = {
      tick: tick,

      restart: function() {
        return stepper.restart(step), simulation;
      },

      stop: function() {
        return stepper.stop(), simulation;
      },
      //设置nodes时会对所有的force进行初始化
      nodes: function(_) {
        return arguments.length ? (nodes = _, initializeNodes(), forces.each(initializeForce), simulation) : nodes;
      },

      alpha: function(_) {
        return arguments.length ? (alpha = +_, simulation) : alpha;
      },

      alphaMin: function(_) {
        return arguments.length ? (alphaMin = +_, simulation) : alphaMin;
      },

      alphaDecay: function(_) {
        return arguments.length ? (alphaDecay = +_, simulation) : +alphaDecay;
      },

      alphaTarget: function(_) {
        return arguments.length ? (alphaTarget = +_, simulation) : alphaTarget;
      },

      velocityDecay: function(_) {
        return arguments.length ? (velocityDecay = 1 - _, simulation) : 1 - velocityDecay;
      },

      force: function(name, _) {
        return arguments.length > 1 ? ((_ == null ? forces.remove(name) : forces.set(name, initializeForce(_))), simulation) : forces.get(name);
      },

      find: function(x, y, radius) {
        var i = 0,
            n = nodes.length,
            dx,
            dy,
            d2,
            node,
            closest;

        if (radius == null) radius = Infinity;
        else radius *= radius;

        for (i = 0; i < n; ++i) {
          node = nodes[i];
          dx = x - node.x;
          dy = y - node.y;
          d2 = dx * dx + dy * dy;
          if (d2 < radius) closest = node, radius = d2;
        }

        return closest;
      },

      on: function(name, _) {
        return arguments.length > 1 ? (event.on(name, _), simulation) : event.on(name);
      }
    };
}
```
上述方法对nodes进行处理，计算其x和y值以及初始化vx、vy值，其中很重要的一部分是force函数，该函数用来模拟物理作用力来改变nodes的位置和速度。

## Force
force函数在定时器执行过程中会重复调用，用于控制nodes的坐标和速度。
### d3.forceLink
forceLink主要用于nodes之间的联系即links，每个link会将两个不同的node以source和target的方式进行连接，同时内部会对vx、vy进行调整。
```
/* d3.forceLink
   * force用于控制节点之间的联系
   * !将于节点相连的link的数量记作该节点的权值
   */
function link(links) {
    var id = index$2,
        strength = defaultStrength,
        strengths,
        //默认link的长度都为30
        distance = constant$6(30),
        distances,
        nodes,
        //count记录跟每个节点有关联的节点数量，即该节点的权值
        count,
        //bias存储每条link对应的source的权值与source和target权值和的比值
        bias,
        iterations = 1;

    if (links == null) links = [];
    //默认计算link的强度的方法
    function defaultStrength(link) {
      return 1 / Math.min(count[link.source.index], count[link.target.index]);
    }

    function force(alpha) {
      for (var k = 0, n = links.length; k < iterations; ++k) {
        for (var i = 0, link, source, target, x, y, l, b; i < n; ++i) {
          link = links[i], source = link.source, target = link.target;
          x = target.x + target.vx - source.x - source.vx || jiggle();
          y = target.y + target.vy - source.y - source.vy || jiggle();
          //target和source的距离为l
          l = Math.sqrt(x * x + y * y);
          l = (l - distances[i]) / l * alpha * strengths[i];
          x *= l, y *= l;
          //对target和source的速度进行调整
          target.vx -= x * (b = bias[i]);
          target.vy -= y * b;
          source.vx += x * (b = 1 - b);
          source.vy += y * b;
        }
      }
    }

    function initialize() {
      if (!nodes) return;

      var i,
          n = nodes.length,
          m = links.length,
          //对nodes中每个值设置id作为键值
          nodeById = map$1(nodes, id),
          link;

      for (i = 0, count = new Array(n); i < n; ++i) {
        count[i] = 0;
      }

      for (i = 0; i < m; ++i) {
        link = links[i], link.index = i;
        //将link中的source和target值作为id来查找node
        if (typeof link.source !== "object") link.source = nodeById.get(link.source);
        if (typeof link.target !== "object") link.target = nodeById.get(link.target);
        ++count[link.source.index], ++count[link.target.index];
      }

      for (i = 0, bias = new Array(m); i < m; ++i) {
        link = links[i], bias[i] = count[link.source.index] / (count[link.source.index] + count[link.target.index]);
      }

      strengths = new Array(m), initializeStrength();
      distances = new Array(m), initializeDistance();
    }

    function initializeStrength() {
      if (!nodes) return;

      for (var i = 0, n = links.length; i < n; ++i) {
        strengths[i] = +strength(links[i], i, links);
      }
    }

    function initializeDistance() {
      if (!nodes) return;

      for (var i = 0, n = links.length; i < n; ++i) {
        distances[i] = +distance(links[i], i, links);
      }
    }

    force.initialize = function(_) {
      nodes = _;
      initialize();
    };

    force.links = function(_) {
      return arguments.length ? (links = _, initialize(), force) : links;
    };

    force.id = function(_) {
      return arguments.length ? (id = _, force) : id;
    };

    force.iterations = function(_) {
      return arguments.length ? (iterations = +_, force) : iterations;
    };

    force.strength = function(_) {
      return arguments.length ? (strength = typeof _ === "function" ? _ : constant$6(+_), initializeStrength(), force) : strength;
    };

    force.distance = function(_) {
      return arguments.length ? (distance = typeof _ === "function" ? _ : constant$6(+_), initializeDistance(), force) : distance;
    };

    return force;
}
```

### d3.forceCenter
forceCenter根据设置的(x, y)坐标而将node的坐标向其移动。
```
//d3.forceCenter
function center$1(x, y) {
    var nodes;

    if (x == null) x = 0;
    if (y == null) y = 0;

    function force() {
      var i,
          n = nodes.length,
          node,
          sx = 0,
          sy = 0;

      for (i = 0; i < n; ++i) {
        node = nodes[i], sx += node.x, sy += node.y;
      }
      //这里简化了粒子的质量，认为都相等，通过sx / n和sy / n得到所有粒子的重心
      for (sx = sx / n - x, sy = sy / n - y, i = 0; i < n; ++i) {
        //将所有粒子的坐标向中心点靠近
        node = nodes[i], node.x -= sx, node.y -= sy;
      }
    }

    force.initialize = function(_) {
      nodes = _;
    };

    force.x = function(_) {
      return arguments.length ? (x = +_, force) : x;
    };

    force.y = function(_) {
      return arguments.length ? (y = +_, force) : y;
    };

    return force;
}
```

### d3.forceCollide
forceCollide方法将nodes不再看做一个点而是一个指定半径的圆形，防止不同的节点发生碰撞即要满足两个圆心的距离大于两个圆形半径之和。
```
//d3.forceCollide
function collide(radius) {
    var nodes,
        radii,
        strength = 1,
        iterations = 1;
    //如果没有设置radius，则默认为1
    if (typeof radius !== "function") radius = constant$6(radius == null ? 1 : +radius);

    function force() {
      var i, n = nodes.length,
          tree,
          node,
          xi,
          yi,
          ri,
          ri2;

      for (var k = 0; k < iterations; ++k) {
        //visitAfter函数使得对每个node都执行prepare。这里采用后续遍历的方法，因为只有知道了孩子节点的半径才能确定根节点半径
        tree = quadtree(nodes, x$1, y$1).visitAfter(prepare);
        //依次访问所有的node节点，判断其他节点是否可能与其重叠
        for (i = 0; i < n; ++i) {
          node = nodes[i];
          ri = radii[i], ri2 = ri * ri;
          xi = node.x + node.vx;
          yi = node.y + node.vy;
          tree.visit(apply);
        }
      }
      //这里用于对重叠的节点进行处理，如果当前节点为根节点则判断node是否与该根节点的范围有重叠，如果没有则返回true，不再访问其子节点；否则继续访问其子节点。
      //如果当前节点为叶子节点，
      function apply(quad, x0, y0, x1, y1) {
        var data = quad.data, rj = quad.r, r = ri + rj;
        if (data) {
          // 只比较index大于i的，可防止重复比较
          if (data.index > i) {
            var x = xi - data.x - data.vx,
                y = yi - data.y - data.vy,
                l = x * x + y * y;
            if (l < r * r) {
              if (x === 0) x = jiggle(), l += x * x;
              if (y === 0) y = jiggle(), l += y * y;
              l = (r - (l = Math.sqrt(l))) / l * strength;
              //根据两个节点间的距离和两个节点的半径对node和data的速度进行调整
              //TODO: 为什么这样调整？？？
              node.vx += (x *= l) * (r = (rj *= rj) / (ri2 + rj));
              node.vy += (y *= l) * r;
              data.vx -= x * (r = 1 - r);
              data.vy -= y * r;
            }
          }
          return;
        }
        return x0 > xi + r || x1 < xi - r || y0 > yi + r || y1 < yi - r;
      }
    }
    //为所有节点设置半径
    function prepare(quad) {
      if (quad.data) return quad.r = radii[quad.data.index];
      //quad是一个数组，即quad不是叶子节点时，将其所有子节点的最大半径复制给quad.r
      for (var i = quad.r = 0; i < 4; ++i) {
        if (quad[i] && quad[i].r > quad.r) {
          quad.r = quad[i].r;
        }
      }
    }
    //初始化时通过radius函数处理nodes得到每个node的半径
    force.initialize = function(_) {
      var i, n = (nodes = _).length; radii = new Array(n);
      for (i = 0; i < n; ++i) radii[i] = +radius(nodes[i], i, nodes);
    };
    //iteration值越大，node节点重叠情况就会越小
    force.iterations = function(_) {
      return arguments.length ? (iterations = +_, force) : iterations;
    };
    //strength用于在两个节点重叠时调整节点的速度
    force.strength = function(_) {
      return arguments.length ? (strength = +_, force) : strength;
    };
    //设置节点的获取半径的函数
    force.radius = function(_) {
      return arguments.length ? (radius = typeof _ === "function" ? _ : constant$6(+_), force) : radius;
    };

    return force;
}
```

### d3.forceManyBody
用于模拟所有粒子间的作用力，例如模拟重力或者静电力。
```
//d3.forceManyBody
function manyBody() {
    var nodes,
        node,
        alpha,
        //当strength为正值时粒子间会互相吸引，当为负值时粒子间会互相排斥
        //在这里表现为当strength为正值时，两个互相作用的粒子速度会增加，互相靠近；为负值时，两个粒子速度减小，互相远离。
        strength = constant$6(-30),
        strengths,
        distanceMin2 = 1,
        distanceMax2 = Infinity,
        //theta用于判断距离远近而采取不同的方法对粒子的速度进行处理
        theta2 = 0.81;

    function force(_) {
      var i, n = nodes.length, tree = quadtree(nodes, x$2, y$2).visitAfter(accumulate);
      for (alpha = _, i = 0; i < n; ++i) node = nodes[i], tree.visit(apply);
    }

    function initialize() {
      if (!nodes) return;
      var i, n = nodes.length;
      strengths = new Array(n);
      for (i = 0; i < n; ++i) strengths[i] = +strength(nodes[i], i, nodes);
    }

    function accumulate(quad) {
      var strength = 0, q, c, x, y, i;

      // 对于根节点，根据其子节点来计算
      if (quad.length) {
        for (x = y = i = 0; i < 4; ++i) {
          if ((q = quad[i]) && (c = q.value)) {
            strength += c, x += c * q.x, y += c * q.y;
          }
        }
        quad.x = x / strength;
        quad.y = y / strength;
      }

      // 对于叶子节点，根据其是否有相同节点来计算strength值
      else {
        q = quad;
        q.x = q.data.x;
        q.y = q.data.y;
        do strength += strengths[q.data.index];
        while (q = q.next);
      }

      quad.value = strength;
    }

    function apply(quad, x1, _, x2) {
      if (!quad.value) return true;

      var x = quad.x - node.x,
          y = quad.y - node.y,
          w = x2 - x1,
          l = x * x + y * y;

      // 如果quad和node间的距离较远则根据value、alpha和l来调整node的速度
      if (w * w / theta2 < l) {
        if (l < distanceMax2) {
          if (x === 0) x = jiggle(), l += x * x;
          if (y === 0) y = jiggle(), l += y * y;
          if (l < distanceMin2) l = Math.sqrt(distanceMin2 * l);
          node.vx += x * quad.value * alpha / l;
          node.vy += y * quad.value * alpha / l;
        }
        return true;
      }

      // 如果quad为根节点则返回去访问其子节点
      else if (quad.length || l >= distanceMax2) return;
      //quad和node相同时不会执行以下过程
      //当quad和node间距离较近时，同时要考虑strength来调整node的速度
      
      if (quad.data !== node || quad.next) {
        if (x === 0) x = jiggle(), l += x * x;
        if (y === 0) y = jiggle(), l += y * y;
        if (l < distanceMin2) l = Math.sqrt(distanceMin2 * l);
      }

      do if (quad.data !== node) {
        w = strengths[quad.data.index] * alpha / l;
        node.vx += x * w;
        node.vy += y * w;
      } while (quad = quad.next);
    }

    force.initialize = function(_) {
      nodes = _;
      initialize();
    };

    force.strength = function(_) {
      return arguments.length ? (strength = typeof _ === "function" ? _ : constant$6(+_), initialize(), force) : strength;
    };

    force.distanceMin = function(_) {
      return arguments.length ? (distanceMin2 = _ * _, force) : Math.sqrt(distanceMin2);
    };

    force.distanceMax = function(_) {
      return arguments.length ? (distanceMax2 = _ * _, force) : Math.sqrt(distanceMax2);
    };

    force.theta = function(_) {
      return arguments.length ? (theta2 = _ * _, force) : Math.sqrt(theta2);
    };

    return force;
}
```

### d3.forceX
使所有的节点向指定的x坐标处靠近。
```
//d3.forceX
function x$3(x) {
    var strength = constant$6(0.1),
        nodes,
        strengths,
        xz;

    if (typeof x !== "function") x = constant$6(x == null ? 0 : +x);

    function force(alpha) {
      for (var i = 0, n = nodes.length, node; i < n; ++i) {
        //通过xz和strength来改变node的x轴方向的速度，使得节点像xz处靠近。strength的值越大，node的速度改变的越快，即会更快的到达指定坐标位置而趋于稳定
        node = nodes[i], node.vx += (xz[i] - node.x) * strengths[i] * alpha;
      }
    }

    function initialize() {
      if (!nodes) return;
      var i, n = nodes.length;
      strengths = new Array(n);
      xz = new Array(n);
      for (i = 0; i < n; ++i) {
        //对每个node分别计算x坐标存入xz数组中，同时计算strength值
        strengths[i] = isNaN(xz[i] = +x(nodes[i], i, nodes)) ? 0 : +strength(nodes[i], i, nodes);
      }
    }

    force.initialize = function(_) {
      nodes = _;
      initialize();
    };

    force.strength = function(_) {
      return arguments.length ? (strength = typeof _ === "function" ? _ : constant$6(+_), initialize(), force) : strength;
    };

    force.x = function(_) {
      return arguments.length ? (x = typeof _ === "function" ? _ : constant$6(+_), initialize(), force) : x;
    };

    return force;
}
```







