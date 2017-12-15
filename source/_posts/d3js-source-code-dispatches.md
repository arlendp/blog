title: d3.js源码分析——Dispatches

categories:
- Web开发
- JS
tags:
- Web开发
- D3.js

date: 2016/09/30
folder: web/js
en-title: d3js-source-code-dispatches
---
d3的dispatch模块是对原生事件处理的封装，通过该模块可以注册自定义的事件并绑定回调函数。
<!--more-->

## d3.dispatch
该模块用于注册自定义名称的回调函数，并且可以调用这些函数。
```
function dispatch() {
    /*将传入的参数作为键值存入Dispatch对象中，参数以数组的形式传入并且不能有重复的元素
     *初始化时参数只包含类型，不应包含名称，并入初始化时可以传入['click', 'drag']，在之后调用on方法时可以通过on('click.my1 drag.my2 hover', callback)这种方式来绑定回调函数
     *在dispatch中是以如下方式存储：
     *
     *  {
     *    'click': [
     *      {
     *        name: 'my1',
     *        value: callback
     *      }
     *    ],
     *    'drag': [
     *      {
     *        name: 'my2',
     *        value: callback
     *      }
     *    ],
     *    'hover': [
     *      {
     *        name: '',
     *        value: callback
     *      }
     *    ]
     *  }
     */
    for (var i = 0, n = arguments.length, _ = {}, t; i < n; ++i) {
        if (!(t = arguments[i] + "") || (t in _)) throw new Error("illegal type: " + t);
        _[t] = [];
    }
    return new Dispatch(_);
}
//Dispatch构造函数
function Dispatch(_) {
    this._ = _;
}

```
将传入的事件类型存入dispatch对象中。

### dispatch.on(typenames[, callback])
用于将事件和回调函数进行绑定。
```
//绑定事件类型和回调函数
  on: function(typename, callback) {
      var _ = this._,
          T = parseTypenames(typename + "", _),
          t,
          i = -1,
          n = T.length;

      // 如果没有callback参数，则返回以指定type和name注册的callback函数。
      if (arguments.length < 2) {
        while (++i < n) if ((t = (typename = T[i]).type) && (t = get(_[t], typename.name))) return t;
        return;
      }

      // 如果传入了callback函数，则对指定的type和name设置该回调函数。
      // 如果callback为null，则可以移除指定的回调函数
      if (callback != null && typeof callback !== "function") throw new Error("invalid callback: " + callback);
      while (++i < n) {
        if (t = (typename = T[i]).type) _[t] = set$1(_[t], typename.name, callback);
        else if (callback == null) for (t in _) _[t] = set$1(_[t], typename.name, null);
      }

      return this;
  },
  
  //获取回调函数
  function get(type, name) {
    for (var i = 0, n = type.length, c; i < n; ++i) {
      if ((c = type[i]).name === name) {
        return c.value;
      }
    }
  }
  //设置回调函数
  function set$1(type, name, callback) {
    for (var i = 0, n = type.length; i < n; ++i) {
      if (type[i].name === name) {
        //如果type中已有指定的name，则将其从type数组中移除
        type[i] = noop$1, type = type.slice(0, i).concat(type.slice(i + 1));
        break;
      }
    }
    if (callback != null) type.push({name: name, value: callback});
    return type;
  }
```

其他方法：
```
//对dispatch进行拷贝，对拷贝后的内容进行修改不会影响之前的内容
copy: function() {
    var copy = {}, _ = this._;
    for (var t in _) copy[t] = _[t].slice();
    return new Dispatch(copy);
},
call: function(type, that) {
    //第二个参数之后的参数会传入callback函数中
    if ((n = arguments.length - 2) > 0) for (var args = new Array(n), i = 0, n, t; i < n; ++i) args[i] = arguments[i + 2];
    if (!this._.hasOwnProperty(type)) throw new Error("unknown type: " + type);
    //会调用type下的所有回调函数
    for (t = this._[type], i = 0, n = t.length; i < n; ++i) t[i].value.apply(that, args);
},
apply: function(type, that, args) {
    if (!this._.hasOwnProperty(type)) throw new Error("unknown type: " + type);
    for (var t = this._[type], i = 0, n = t.length; i < n; ++i) t[i].value.apply(that, args);
}
```





