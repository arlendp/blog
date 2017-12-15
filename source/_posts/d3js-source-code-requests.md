title: d3.js源码分析——Requests

categories:
- Web开发
- JS
tags:
- Web开发
- D3.js

date: 2016/10/07
folder: web/js
en-title: d3js-source-code-requests
---
文件的加载对于很多应用都格外重要，对d3来说也是如此。对于绘制结构简单、数据量不大的图形，尚且可以将数据和js代码存放在一起，但是对于数据结构复杂、数据量庞大的情况，数据集应该被单独存放于独立的文件中，因此这里就涉及到了文件的读取问题。该模块用于对原生的`XMLHttpRequest`进行封装，可用于加载文件。
<!--more-->

## d3.request

```
// d3.request
function request(url, callback) {
    var request,
        event = dispatch("beforesend", "progress", "load", "error"),
        mimeType,
        headers = map$1(),
        xhr = new XMLHttpRequest,
        user = null,
        password = null,
        response,
        responseType,
        timeout = 0;

    // 对于不支持跨域资源共享的IE浏览器，采用XDomainRequest
    if (typeof XDomainRequest !== "undefined"
        && !("withCredentials" in xhr)
        && /^(http(s)?:)?\/\//.test(url)) xhr = new XDomainRequest;
    //如果支持onload回调方法则绑定该方法，否则使用onreadystatechange方法
    "onload" in xhr
        //三种事件分别是请求成功完成、请求失败和请求时限到期未完成
        ? xhr.onload = xhr.onerror = xhr.ontimeout = respond
        : xhr.onreadystatechange = function(o) { xhr.readyState > 3 && respond(o); };

    function respond(o) {
      var status = xhr.status, result;
      if (!status && hasResponse(xhr)
          || status >= 200 && status < 300
          || status === 304) {
        if (response) {
          try {
            //先通过response函数对返回结果进行处理
            result = response.call(request, xhr);
          } catch (e) {
            event.call("error", request, e);
            return;
          }
        } else {
          result = xhr;
        }
        event.call("load", request, result);
      } else {
        event.call("error", request, o);
      }
    }

    xhr.onprogress = function(e) {
      event.call("progress", request, e);
    };

    request = {
      //设置请求头信息
      header: function(name, value) {
        name = (name + "").toLowerCase();
        if (arguments.length < 2) return headers.get(name);
        if (value == null) headers.remove(name);
        else headers.set(name, value + "");
        return request;
      },

      // 设置服务器返回数据类型，用于accept请求头和overrideMimeType方法
      mimeType: function(value) {
        if (!arguments.length) return mimeType;
        mimeType = value == null ? null : value + "";
        return request;
      },

      /* 指定返回值类型，可以为以下几种值：
       * ”“：字符串（默认值）
       * “arraybuffer”：ArrayBuffer对象
       * “blob”：Blob对象
       * “document”：Document对象
       * “json”：JSON对象
       * “text”：字符串
       */
      responseType: function(value) {
        if (!arguments.length) return responseType;
        responseType = value;
        return request;
      },

      timeout: function(value) {
        if (!arguments.length) return timeout;
        timeout = +value;
        return request;
      },

      user: function(value) {
        return arguments.length < 1 ? user : (user = value == null ? null : value + "", request);
      },

      password: function(value) {
        return arguments.length < 1 ? password : (password = value == null ? null : value + "", request);
      },

      //将返回内容转化成指定类型
      response: function(value) {
        response = value;
        return request;
      },

      // 使用GET方法发送请求
      get: function(data, callback) {
        return request.send("GET", data, callback);
      },

      // 使用POST方法发送请求
      post: function(data, callback) {
        return request.send("POST", data, callback);
      },

      // 修改请求头信息，设置服务器返回数据类型，监听error和load事件并设置回调函数
      send: function(method, data, callback) {
        xhr.open(method, url, true, user, password);
        if (mimeType != null && !headers.has("accept")) headers.set("accept", mimeType + ",*/*");
        //设置请求头
        if (xhr.setRequestHeader) headers.each(function(value, name) { xhr.setRequestHeader(name, value); });
        //指定服务器返回数据类型
        if (mimeType != null && xhr.overrideMimeType) xhr.overrideMimeType(mimeType);
        if (responseType != null) xhr.responseType = responseType;
        if (timeout > 0) xhr.timeout = timeout;
        if (callback == null && typeof data === "function") callback = data, data = null;
        if (callback != null && callback.length === 1) callback = fixCallback(callback);
        if (callback != null) request.on("error", callback).on("load", function(xhr) { callback(null, xhr); });
        //调用beforesend监听事件
        event.call("beforesend", request, xhr);
        //发送请求
        xhr.send(data == null ? null : data);
        return request;
      },

      abort: function() {
        xhr.abort();
        return request;
      },
      //设置事件监听函数，只能是以下类型：beforesend、progress、load和error
      on: function() {
        var value = event.on.apply(event, arguments);
        return value === event ? request : value;
      }
    };
    //如果callback参数被传入，则会立即将请求发送出去，若没有传入callback则可继续配置request
    if (callback != null) {
      if (typeof callback !== "function") throw new Error("invalid callback: " + callback);
      return request.get(callback);
    }

    return request;
}
```

## d3.csv
用于读取指定URL中的csv文件。
```
//d3.csv
var csv$1 = dsv$1("text/csv", csvParse);

function dsv$1(defaultMimeType, parse) {
    return function(url, row, callback) {
      //可以省略row函数
      if (arguments.length < 3) callback = row, row = null;
      var r = request(url).mimeType(defaultMimeType);
      //设置row函数
      r.row = function(_) { return arguments.length ? r.response(responseOf(parse, row = _)) : row; };
      r.row(row);
      return callback ? r.get(callback) : r;
    };
}
  
//返回解析函数
function responseOf(parse, row) {
    return function(request) {
      return parse(request.responseText, row);
    };
}
  
```
这部分是直接调用request方法来读取文件，修改了服务器返回数据类型和response方法。与以下方法等同：
```
d3.request(url)
    .mimeType("text/csv")
    .response(function(xhr) { return d3.csvParse(xhr.responseText, row); })
    .get(callback);
```

## d3.html
用于读取HTML文件。
```
var html = type("text/html", function(xhr) {
    return document.createRange().createContextualFragment(xhr.responseText);
});
```
将返回的字符串构造成document fragment，形成一个DOM节点可以对其进行操作。

## d3.json
读取JSON文件。
```
//d3.json
var json = type("application/json", function(xhr) {
    return JSON.parse(xhr.responseText);
});
```
通过`JSON.parse`方法对返回的字符串进行处理，转化成json格式的数据。

## d3.tsv
用于读取tsv文件，和上述`d3.csv`类似。
等同于：
```
d3.request(url)
    .mimeType("text/tab-separated-values")
    .response(function(xhr) { return d3.tsvParse(xhr.responseText, row); })
    .get(callback);
```

