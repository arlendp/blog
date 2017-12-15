title: d3.js源码分析——Selections

categories:
- Web开发
- JS
tags:
- Web开发
- D3.js

date: 2016/08/30
folder: web/js
en-title: d3js-source-code-selections
---
在web开发中我们会花大量的时间用于dom的操作上，一般情况下我们会选择第三方库如jQuery来替代原生的方法，因为原生的方法在操作上会使代码大量重复冗余而不易操作。d3同样提供了一套自己的方法来很方便的对dom进行操作，像修改样式、注册时间等等都可以通过它来完成。
<!--more-->

d3的selection主要用于直接对DOM进行操作，如设置属性、修改样式等等，同时它可以和data join（数据连接）这一强大的功能结合起来对元素和元素上绑定的数据进行操作。
selection中的方法计算后返回当前selection，这样可以进行方法的链式调用。由于通过这种方式调用方法会使得每行的代码很长，因此按照约定：若该方法返回的是**当前的selection**，则使用**四个空格**进行缩进；若方法返回的是**新的selection**，则使用**两个空格**进行缩进。
## 选择元素
元素的选择通过两种方法`select`和`selectAll`来实现，前者只返回第一个匹配的元素，而后者返回所有匹配元素。
由于之后所有的操作都是在selection上进行，而该对象则是通过Selection构造函数得到的，源码如下：
```
function Selection(groups, parents) {
    this._groups = groups;
    this._parents = parents;
}
```
可以看出，selection对象包含两个基本属性`_groups`和`_parents`，前者用于存储结点组，而后者则存储结点的父节点信息。
### selection.select(selector)
```
var p = d3.selectAll('div')
          .select('p');
```
该方法对selection中的每个元素进行查找，选择其中第一个匹配selector的子元素，源码如下：
```
/*
 * Selection的select方法
 * 通过select方法选择时，若不存在元素，则会在数组中将该位置留出（赋值为null）用于之后插入时使用。
 */

function selection_select(select) {
    if (typeof select !== "function") select = selector(select);
    //当select是函数时，直接调用该函数，并依次对该函数传入data信息、当前的索引和当前的结点group，同时将函数的this设置为当前dom对象
    for (var groups = this._groups, m = groups.length, subgroups = new Array(m), j = 0; j < m; ++j) {
        for (var group = groups[j], n = group.length, subgroup = subgroups[j] = new Array(n), node, subnode, i = 0; i < n; ++i) {
            if ((node = group[i]) && (subnode = select.call(node, node.__data__, i, group))) {
          //当node中有data信息时，node的子元素也添加该data信息
            if ("__data__" in node) subnode.__data__ = node.__data__;
                subgroup[i] = subnode;
            }
        }
    }
    //将父级的_parents属性作为子元素的_parents属性
    return new Selection(subgroups, this._parents);
}
```
若selector为选择器字符串时，则会先调用selector方法将其转化为函数，源码如下：
```
function selector(selector) {
    return selector == null ? none$2 : function() {
        //只返回第一个选中元素
        return this.querySelector(selector);
    };
}
```
可以看出，其内部调用的是js的原生方法`querySelector`。
上述代码对select参数进行处理后，使得其转化为函数，在后来的循环中调用时，通过`select.call`进行调用，传入的参数依次为**结点的__data__属性值，结点在该结点组中的索引，该结点组**。
有以下几点值得注意：
* 若结点中包含`__data__`属性则会对匹配的子元素也设置该属性。
* 通过select方法得到的新的selection的`_parents`值并不会改变。

### selection.selectAll(selector)
```
var p = d3.selectAll('div')
          .selectAll('p');
```
该方法对selection中的每个元素进行查找，选择其中匹配selector的子元素，返回的selection中的元素根据其父结点进行对应的分组，源码如下：
```
/*
 * 对selection进行selectAll计算会改变selection结构，parents也会改变
 */
function selection_selectAll(select) {
    if (typeof select !== "function") select = selectorAll(select);
    for (var groups = this._groups, m = groups.length, subgroups = [], parents = [], j = 0; j < m; ++j) {
        for (var group = groups[j], n = group.length, node, i = 0; i < n; ++i) {
            if (node = group[i]) {
                subgroups.push(select.call(node, node.__data__, i, group));
                parents.push(node);
            }
        }
    }
    return new Selection(subgroups, parents);
}
```
上述代码可以看出，对node调用select方法后，查找到的结果存入subgroups中，同时将node作为父结点存入parents数组中，使得结点与父结点一一对应，最终返回新的selection。

### selection.filter(filter)
```
var red = d3.selectAll('p')
            .filter('.red')
```
将使得filter为true的元素构造成新的selection并返回，源码如下：
```
//filter方法对当前selection进行过滤，保留满足条件的元素
function selection_filter(match) {
    if (typeof match !== "function") match = matcher$1(match);
    for (var groups = this._groups, m = groups.length, subgroups = new Array(m), j = 0; j < m; ++j) {
            for (var group = groups[j], n = group.length, subgroup = subgroups[j] = [], node, i = 0; i < n; ++i) {
                if ((node = group[i]) && match.call(node, node.__data__, i, group)) {
                subgroup.push(node);
            }
        }
    }
    return new Selection(subgroups, this._parents);
}
```
若`match`不为函数，则通过`matcher$1`函数对其进行处理，其源码如下：
```
var matcher = function(selector) {
    return function() {
        //Element.matches(s)，如果元素能通过s选择器选择到则返回true；否则返回false
        return this.matches(selector);
    };
};
```
可看出`matcher`函数内部是调用原生的`Element.matches`方法实现。

### selection.merge(other_selection)
```
var circle = svg.selectAll("circle").data(data) // UPDATE
    .style("fill", "blue");

circle.exit().remove(); // EXIT

circle.enter().append("circle") // ENTER
    .style("fill", "green")
  .merge(circle) // ENTER + UPDATE
    .style("stroke", "black");
```
该方法将两个selection进行合并成一个新的selection并返回，源码如下：
```
function selection_merge(selection) {
    //新的selection的_.groups长度和groups0相同，合并时只在m范围内计算
    for (var groups0 = this._groups, groups1 = selection._groups, m0 = groups0.length, m1 = groups1.length, m = Math.min(m0, m1), merges = new Array(m0), j = 0; j < m; ++j) {
        for (var group0 = groups0[j], group1 = groups1[j], n = group0.length, merge = merges[j] = new Array(n), node, i = 0; i < n; ++i) {
            //groups0数组大小不变，只有在group0[i]不存在，group1[i]存在时才选择group1[i]
            if (node = group0[i] || group1[i]) {
                merge[i] = node;
            }
        }
    }
    // 若m1 < m0，则将groups0剩余的复制过来
    for (; j < m0; ++j) {
      merges[j] = groups0[j];
    }
    return new Selection(merges, this._parents);
}
```
该方法实际上相当于对this selection中的空元素进行填充。

## 修改元素
在选择元素之后可以使用selection的方法来修改元素，如样式、属性等。

### selection.attr(name[, value])
```
var p = d3.selectAll('p')
            .attr('class', 'red');
```
对selection中的元素以指定的name和value设置属性，并返回当前selection，源码如下：
```
function selection_attr(name, value) {
    var fullname = namespace(name);

    if (arguments.length < 2) {
        //得到selection中第一个存在的元素
        var node = this.node();
        return fullname.local
            ? node.getAttributeNS(fullname.space, fullname.local)
            : node.getAttribute(fullname);
    }

    return this.each((value == null
        ? (fullname.local ? attrRemoveNS : attrRemove) : (typeof value === "function"
        ? (fullname.local ? attrFunctionNS : attrFunction)
        : (fullname.local ? attrConstantNS : attrConstant)))(fullname, value));
}
```
若只有name参数时，则返回第一个存在的元素的name属性值，调用的是原生的`Element.getAttribute`方法。
当有两个参数时，调用`selection.each`方法对selection中的每个元素进行操作，其源码如下：
```
function selection_each(callback) {
    for (var groups = this._groups, j = 0, m = groups.length; j < m; ++j) {
        for (var group = groups[j], i = 0, n = group.length, node; i < n; ++i) {
            if (node = group[i]) callback.call(node, node.__data__, i, group);
        }
    }
    return this;
}
```
若value值为函数，则将name和value传入attrFunction函数中进行处理。
```
function attrFunction(name, value) {
    return function() {
        var v = value.apply(this, arguments);
        if (v == null) this.removeAttribute(name);
        else this.setAttribute(name, v);
    };
}
```
在`selection.each`方法中，将参数传入value函数中，根据value返回结果选择设置和删除属性操作。

### selection.classed(names[, value])
```
var p = d3.selectAll('p')
            .classed('red warn', true);
```
对selection中的元素设置类名，源码如下：
```
// 当value为真值时，在所有元素类名中添加name；否则删除name
function selection_classed(name, value) {
    var names = classArray(name + "");
    // 当只有name参数时，判断该selection对象的_groups里第一个存在的结点是否包含所有的name的类名，如果是则返回true；否则，返回false
    if (arguments.length < 2) {
        var list = classList(this.node()), i = -1, n = names.length;
        while (++i < n) if (!list.contains(names[i])) return false;
        return true;
    }
    return this.each((typeof value === "function"
        ? classedFunction : value
        ? classedTrue
        : classedFalse)(names, value));
}
```
其中`classArray`方法是将类名字符串拆分成数组：
```
// 将类名拆分成数组，如'button button-warn' => ['button', 'button-warn']
function classArray(string) {
    return string.trim().split(/^|\s+/);
}
```

### selection.style(name[, value[, priority]])
```
var p = d3.selectAll('p')
            .style('color', 'red');
```
该方法对selection中的元素设置样式，源码如下：
```
// 设置selection的样式，注意样式的单位问题
function selection_style(name, value, priority) {
    var node;
    return arguments.length > 1
        ? this.each((value == null
              ? styleRemove : typeof value === "function"
              ? styleFunction
              : styleConstant)(name, value, priority == null ? "" : priority))
        : window(node = this.node())
            .getComputedStyle(node, null)
            .getPropertyValue(name);
}
```
从上述代码可以看出，获取样式是通过`window.getComputedStyle(element).getPropertyValue(name)`来得到（该方法得到的值是只读的），而删除样式则是通过`element.style.removeProperty(name)`来实现，设置属性通过`element.style.setProperty(name, value)`来实现。

### selection.property(name[, value])
```
var checkbox = d3.selectAll('input[type=checkbox]')
                    .property('checked', 'checked');
```
该方法设置一些特殊的属性。
```
function selection_property(name, value) {
    return arguments.length > 1
        ? this.each((value == null
            ? propertyRemove : typeof value === "function"
            ? propertyFunction
            : propertyConstant)(name, value))
        : this.node()[name];
}

function propertyRemove(name) {
    return function() {
      delete this[name];
    };
  }

function propertyConstant(name, value) {
    return function() {
      this[name] = value;
    };
}

function propertyFunction(name, value) {
    return function() {
      var v = value.apply(this, arguments);
      if (v == null) delete this[name];
      else this[name] = v;
    };
}
```
内部通过直接修改元素的属性来实现。

### selection.text([value])
该方法对selection中的所有元素设置文本内容，同时会替换掉元素中的子元素，源码如下：
```
function textRemove() {
    this.textContent = "";
}

function textConstant(value) {
    return function() {
      this.textContent = value;
    };
}

function textFunction(value) {
    return function() {
      var v = value.apply(this, arguments);
      this.textContent = v == null ? "" : v;
    };
}
// 设置元素的textContent属性，该属性返回的是元素内的纯文本内容，不包含结点标签（但包含标签内的文本）
function selection_text(value) {
    return arguments.length
        ? this.each(value == null
            ? textRemove : (typeof value === "function"
            ? textFunction
            : textConstant)(value))
        : this.node().textContent;
}
```
上述代码是通过`element.textContent`方法来获取和修改文本内容。

### selection.html([value])
对selection中的所有元素设置innerHTML。方法同上述`selection.text`类似，只是通过`element.innerHTML`来修改元素内的所有内容。

### selection.append(type)
对selection中的元素添加新的元素。
```
/*
 * selection的append方法
 * 该方法返回新的Selection对象，
 */
function selection_append(name) {
    var create = typeof name === "function" ? name : creator(name);
    return this.select(function() {
        //arguments是传入当前匿名函数的参数
        return this.appendChild(create.apply(this, arguments));
    });
}

function creatorInherit(name) {
    return function() {
        //ownerDocument返回当前document对象
        var document = this.ownerDocument,
            uri = this.namespaceURI;
        return uri === xhtml && document.documentElement.namespaceURI === xhtml
            ? document.createElement(name)//创建dom对象
            : document.createElementNS(uri, name);
    };
}
```
该方法中调用到了`selection.select`方法，由于`element.appendChild`方法返回的是该子结点，因此返回的新的selection包含的是所有添加的子结点。

### selection.insert(type, before)
对selection中的元素插入新的元素，同上述`selection.append`方法类似，只是内部使用`element.insertBefore`方法实现。

### selection.sort(compare)
根据`compare`函数对selection中的元素进行排序，排好序后按照排序结果对dom进行排序，返回排序后新创建的selection对象。
```
function selection_sort(compare) {
    if (!compare) compare = ascending$2;

    function compareNode(a, b) {
      // 比较结点的data大小
      return a && b ? compare(a.__data__, b.__data__) : !a - !b;
    }
    // copy一份selection中的_groups包含的结点
    for (var groups = this._groups, m = groups.length, sortgroups = new Array(m), j = 0; j < m; ++j) {
        for (var group = groups[j], n = group.length, sortgroup = sortgroups[j] = new Array(n), node, i = 0; i < n; ++i) {
            if (node = group[i]) {
                sortgroup[i] = node;
            }
        }
        // 调用array的sort方法
        sortgroup.sort(compareNode);
    }
    return new Selection(sortgroups, this._parents).order();
}

// 递增
function ascending$2(a, b) {
    return a < b ? -1 : a > b ? 1 : a >= b ? 0 : NaN;
}
```
若没有compare参数，则默认以递增的方式排序。同时排序是首先比较元素中的`__data__`属性值的大小，对selection排好序后调用order方法。

### selection.sort(compare)
按照selection中每组内元素的顺序对dom进行排序
```
// 对dom结点进行排序
function selection_order() {
    for (var groups = this._groups, j = -1, m = groups.length; ++j < m;) {
        for (var group = groups[j], i = group.length - 1, next = group[i], node; --i >= 0;) {
            if (node = group[i]) {
                // 将node移至next的前面，并将node赋值给next
                if (next && next !== node.nextSibling) next.parentNode.insertBefore(node, next);
                next = node;
            }
        }
    }
    return this;
}
```

## 连接数据
连接数据是将数据绑定到selection对象上，实际上是将数据存储到`__data__`属性中，这样之后对selection的操作过程中便可以直接使用绑定好的数据。主要要理解`update`、`enter`和`exit`，可参考文章[Thinking With Joins](https://bost.ocks.org/mike/join/)。

### selection.data([data[, key]])
该方法将指定的data数组绑定到选中的元素上，返回的selection包含成功绑定数据的元素，也叫做`updata selection`，源码如下：
```
function selection_data(value, key) {
    //当value为假值时，将selection所有元素的__data__属性以数组形式返回
    if (!value) {
      data = new Array(this.size()), j = -1;
      this.each(function(d) { data[++j] = d; });
      return data;
    }

    var bind = key ? bindKey : bindIndex,
        parents = this._parents,
        groups = this._groups;

    if (typeof value !== "function") value = constant$4(value);

    for (var m = groups.length, update = new Array(m), enter = new Array(m), exit = new Array(m), j = 0; j < m; ++j) {
      var parent = parents[j],
          group = groups[j],
          groupLength = group.length,
          data = value.call(parent, parent && parent.__data__, j, parents),
          dataLength = data.length,
          enterGroup = enter[j] = new Array(dataLength),
          updateGroup = update[j] = new Array(dataLength),
          exitGroup = exit[j] = new Array(groupLength);

      bind(parent, group, enterGroup, updateGroup, exitGroup, data, key);

      // 对enter结点设置_next属性，存储其索引之后的第一个update结点
      for (var i0 = 0, i1 = 0, previous, next; i0 < dataLength; ++i0) {
        if (previous = enterGroup[i0]) {
          if (i0 >= i1) i1 = i0 + 1;
          while (!(next = updateGroup[i1]) && ++i1 < dataLength);
          previous._next = next || null;
        }
      }
    }
    // 将enter和exit存入update selection的属性中
    update = new Selection(update, parents);
    update._enter = enter;
    update._exit = exit;
    return update;
}
```
通过上述代码可以看到，对每组group绑定的是相同的data数据。
当没有key参数时，绑定数据使用的是`bindIndex`方法，按照索引一次绑定。
```
function bindIndex(parent, group, enter, update, exit, data) {
    var i = 0,
        node,
        groupLength = group.length,
        dataLength = data.length;

    /*
     * 将data数据绑定到node，并将该node存入update数组中
     * 将剩余的data数据存入enter数组中
     */
    for (; i < dataLength; ++i) {
      if (node = group[i]) {
        node.__data__ = data[i];
        update[i] = node;
      } else {
        enter[i] = new EnterNode(parent, data[i]);
      }
    }

    // 将剩余的node存入exit数组中
    for (; i < groupLength; ++i) {
      if (node = group[i]) {
        exit[i] = node;
      }
    }
  }
```
若含有key参数，则调用`bindKey`方法来绑定数据。
```
function bindKey(parent, group, enter, update, exit, data, key) {
    var i,
        node,
        nodeByKeyValue = {},
        groupLength = group.length,
        dataLength = data.length,
        keyValues = new Array(groupLength),
        keyValue;

    // 对group中每个结点计算keyValue，如果之后的结点含有与前面结点相同的keyValue则将该结点存入exit数组中
    for (i = 0; i < groupLength; ++i) {
      if (node = group[i]) {
        keyValues[i] = keyValue = keyPrefix + key.call(node, node.__data__, i, group);
        if (keyValue in nodeByKeyValue) {
          exit[i] = node;
        } else {
          nodeByKeyValue[keyValue] = node;
        }
      }
    }

    // 对每一个data计算keyValue，如果该keyValue已存在nodeByKeyValue数组中，则将其对应的node存入update数组且绑定data数据；否则将data存入enter中
    for (i = 0; i < dataLength; ++i) {
      keyValue = keyPrefix + key.call(parent, data[i], i, data);
      if (node = nodeByKeyValue[keyValue]) {
        update[i] = node;
        node.__data__ = data[i];
        nodeByKeyValue[keyValue] = null;
      } else {
        enter[i] = new EnterNode(parent, data[i]);
      }
    }

    // 将剩余的没有绑定数据的结点存入exit数组
    for (i = 0; i < groupLength; ++i) {
      if ((node = group[i]) && (nodeByKeyValue[keyValues[i]] === node)) {
        exit[i] = node;
      }
    }
}
```

### selection.enter()
返回selections中的enter selection，即`selection._enter`的结果。
```
function sparse(update) {
    return new Array(update.length);
  }
// selection的enter方法
function selection_enter() {
    return new Selection(this._enter || this._groups.map(sparse), this._parents);
}
```
若selection没有`_enter`属性，即没有进行过`data`操作，则创建空的数组。

### selection.exit()
返回selection中的exit selection，即`selection._exit`的结果。
```
function selection_exit() {
    return new Selection(this._exit || this._groups.map(sparse), this._parents);
}
```

### selection.datum([value])
对selection中的每个元素设置绑定数据，该方法并不会影响到`enter`和`exit`的值。
```
function selection_datum(value) {
    return arguments.length
        ? this.property("__data__", value)
        : this.node().__data__;
}
```
可见其调用`selection.property`方法来设置`__data__`属性。
经常会使用该方法来进行HTML5 data属性的访问，如
```
selection.datum(function() {return this.dataset});
```
`element.dataset`是原生方法，返回的是元素绑定的所有data属性。

## 处理事件

### selection.on(typenames[, listener[, capture]])
该方法用于对selection中的元素添加或者移除事件。
```
function selection_on(typename, value, capture) {
    var typenames = parseTypenames$1(typename + ""), i, n = typenames.length, t;
    // 如果只有typename参数，根据type和name值来找到selection中第一个存在的元素的__on属性中对应的value值。
    if (arguments.length < 2) {
      var on = this.node().__on;
      if (on) for (var j = 0, m = on.length, o; j < m; ++j) {
        for (i = 0, o = on[j]; i < n; ++i) {
          if ((t = typenames[i]).type === o.type && t.name === o.name) {
            return o.value;
          }
        }
      }
      return;
    }
    // 如果value为真值，则添加事件；否则移除事件。
    on = value ? onAdd : onRemove;
    if (capture == null) capture = false;
    for (i = 0; i < n; ++i) this.each(on(typenames[i], value, capture));
    return this;
  }
```
首先是对typename参数进行处理，使用的是`parseTypenames`函数。
```
// 对typenames根据空格来划分成数组，并根据分离后的字符串中的'.'来将该字符串分割为type和name部分，如：'click.foo click.bar' => [{type: 'click', name: 'foo'}, {type: 'click', name: 'bar'}]
function parseTypenames$1(typenames) {
    return typenames.trim().split(/^|\s+/).map(function(t) {
      var name = "", i = t.indexOf(".");
      if (i >= 0) name = t.slice(i + 1), t = t.slice(0, i);
      return {type: t, name: name};
    });
}
```
根据value是否为真值来选择添加或者移除事件。
```
// 添加事件函数
function onAdd(typename, value, capture) {
    var wrap = filterEvents.hasOwnProperty(typename.type) ? filterContextListener : contextListener;
    return function(d, i, group) {
        var on = this.__on, o, listener = wrap(value, i, group);
        if (on) for (var j = 0, m = on.length; j < m; ++j) {
            // 如果新事件的type和name和之前已绑定的事件相同，则移除之前的事件并绑定新的事件
            if ((o = on[j]).type === typename.type && o.name === typename.name) {
                this.removeEventListener(o.type, o.listener, o.capture);
                this.addEventListener(o.type, o.listener = listener, o.capture = capture);
                o.value = value;
                return;
            }
        }
        // 添加事件，并将事件信息存入selection.__on属性中
        this.addEventListener(typename.type, listener, capture);
        o = {type: typename.type, name: typename.name, value: value, listener: listener, capture: capture};
        if (!on) this.__on = [o];
        else on.push(o);
    };
}
```
```
// 移除事件函数
function onRemove(typename) {
    return function() {
        var on = this.__on;
        if (!on) return;
        for (var j = 0, i = -1, m = on.length, o; j < m; ++j) {
            if (o = on[j], (!typename.type || o.type === typename.type) && o.name === typename.name) {
            //对结点移除事件
            this.removeEventListener(o.type, o.listener, o.capture);
            } else {
                //修改结点的__on属性值
                on[++i] = o;
            }
        }
        if (++i) on.length = i;
        //on为空则删除__on属性
        else delete this.__on;
    };
}
```

### selection.dispatch(type[, parameters]) 
对selection中的元素分派指定类型的自定义事件，其中parameters可能包含以下内容：
* bubbles：设置为true表示事件可以冒泡
* cancelable：设置为true表示事件可以被取消
* detail：绑定到事件上的自定义数据
源码如下：
```
//分派事件
function selection_dispatch(type, params) {
    return this.each((typeof params === "function"
        ? dispatchFunction
        : dispatchConstant)(type, params));
}

function dispatchConstant(type, params) {
    return function() {
      return dispatchEvent(this, type, params);
    };
}

function dispatchFunction(type, params) {
    return function() {
      return dispatchEvent(this, type, params.apply(this, arguments));
    };
}
```
`dispatchEvent`函数如下：
```
//创建自定义事件并分派给指定元素
function dispatchEvent(node, type, params) {
    var window$$ = window(node),
        event = window$$.CustomEvent;

    if (event) {
      event = new event(type, params);
    } else {
        //该方法已被废弃
        event = window$$.document.createEvent("Event");
        if (params) event.initEvent(type, params.bubbles, params.cancelable), event.detail = params.detail;
        else event.initEvent(type, false, false);
    }

    node.dispatchEvent(event);
}
```
### d3.event
存储当前事件，在调用事件监听器的时候设置，处理函数执行完毕后重置，可以获取其中包含的事件信息如：`event.pageX`。

### d3.customEvent(event, listener[, that[, arguments]])
该方法调用指定的监听器。
```
//调用指定的监听器
function customEvent(event1, listener, that, args) {
    //记录当前事件
    var event0 = exports.event;
    event1.sourceEvent = exports.event;
    exports.event = event1;
    try {
        return listener.apply(that, args);
    } finally {
        //监听器执行完后恢复事件
        exports.event = event0;
    }
}
```

### d3.mouse(container)
返回当前当前事件相对于指定容器的x和y坐标。
```
function mouse(node) {
    var event = sourceEvent();
    //如果是触摸事件，返回Touch对象
    if (event.changedTouches) event = event.changedTouches[0];
    return point$5(node, event);
}
```
`point$5`方法用于计算坐标。
```
function point$5(node, event) {
    //若node是svg元素，则获取svg容器
    var svg = node.ownerSVGElement || node;

    if (svg.createSVGPoint) {
      //创建SVGPoint对象
      var point = svg.createSVGPoint();
      //将事件相对客户端的x和y坐标赋值给point对象
      point.x = event.clientX, point.y = event.clientY;
      //进行坐标转换
      point = point.matrixTransform(node.getScreenCTM().inverse());
      return [point.x, point.y];
    }

    var rect = node.getBoundingClientRect();
    //返回event事件相对于容器的坐标
    return [event.clientX - rect.left - node.clientLeft, event.clientY - rect.top - node.clientTop];
}
```

## 控制流
用于selection的一些高级操作。
### selection.each(function)
为每一个选中的元素调用指定的函数。
```
//selection的each方法
function selection_each(callback) {

    for (var groups = this._groups, j = 0, m = groups.length; j < m; ++j) {
      for (var group = groups[j], i = 0, n = group.length, node; i < n; ++i) {
        //通过call方式调用回调函数
        if (node = group[i]) callback.call(node, node.__data__, i, group);
        }
    }
    return this;
}
```

### selection.call(function[, arguments…])
将selection和其他参数传入指定函数中执行，并返回当前selection，这和直接链式调用`function(selection)`相同。
```
function selection_call() {
    var callback = arguments[0];
    arguments[0] = this;
    callback.apply(null, arguments);
    return this;
}
```

## 局部变量
d3的局部变量可以定义与data独立开来的局部状态，它的作用域是dom元素。
其构造函数和原型方法如下：
```
function Local() {
    this._ = "@" + (++nextId).toString(36);
}

Local.prototype = local.prototype = {
    constructor: Local,
    get: function(node) {
      var id = this._;
      while (!(id in node)) if (!(node = node.parentNode)) return;
      return node[id];
    },
    set: function(node, value) {
      return node[this._] = value;
    },
    remove: function(node) {
      return this._ in node && delete node[this._];
    },
    toString: function() {
      return this._;
    }
};
```