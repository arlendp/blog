title: Webpack打包优化

categories:
- Web开发
- JS
tags:
- Web开发
- JS

date: 2017/01/10
folder: web/js
en-title: webpack-packaging-optimization
---
前端的打包工具从之前的browserify、grunt、gulp到现如今的rollup、webpack，涌现出了很多优秀的打包工具，而目前最火的无疑是webpack，无论是当前热门的框架还是工具库很多都选择了它作为打包工具，因此在开发中webpack作为打包工具是一个很好的选择。在最近的项目开发中我也用到了webpack，其中也碰到了不少优化方面的问题，这里总结一下webpack打包优化的一些细节和方法。

<!--more-->

首先，这次项目用到的是vue的全家桶，在webpack的配置方面直接用的是`vue-cli`生成的默认配置，项目打包完成后发现生成的`vendor.js`文件体积特别大，其次打包过程相当缓慢，因此想尝试各种方式对其进行优化。

## 定位体积大的模块

要想对打包体积进行优化，首先得找到体积大的模块，在这里我们可以使用webpack插件`webpack-bundle-analyzer`来查看整个项目的体积结构对比，它是以treemap的形式展现出来，很形象直观，还有一些具体的交互形式。既可以查看你项目中用到的所有依赖，也可以直观看到各个模块体积在整个项目中的占比。

![webpack-bundle-analyzer](https://cloud.githubusercontent.com/assets/302213/20628702/93f72404-b338-11e6-92d4-9a365550a701.gif)

该插件的使用方法可以直接通过`npm install webpack-bundle-analyzer --save-dev`安装，并在webpack的配置信息中的`plugins: [new BundleAnalyzerPlugin()]`中添加即可。对于`vue-cli`中的配置方式，默认是安装了该插件，但是没有启用，找到`config/index.js`文件在`build`下面会有`bundleAnalyzerReport`的配置，默认是`process.env.npm_config_report`，这里建议在`package.json`的`scripts`中添加一行`"analyz": "npm_config_report=true npm run build"`，这样每次想启用该插件时只需要`npm run analyze`即可。

## 提取公共模块

对于webpack，它在模块化打包上有两点是其核心功能，一是它支持大量的模块类型，无论是`TypeScript`、`CoffeeScript`还是`sass`、`stylus`等语言它都支持，二是它可以通过配置来控制打包文件的粒度，这个下面会讲到。

在开发中我们往往会将所有的依赖库单独提取出来，而不与我们的项目代码混在一起，这时我们会用到一个webpack自带的插件`CommonsChunkPlugin`，从名字上就可以看出它是一个提取公共模块的插件。从它的文档中可以看出可以传入一个对象最为参数，在使用中常用的三个参数分别为：

* name(names)
* minChunks
* chunks

`name`好理解，指的就是最后打包文件的名字，而如果使用的是`names`的话，传入的必须是一个字符串数组。`minChunks`如果传入的是一个数字的话，指的是如果该模块被其他模块的引用次数达到了这个数值的话该模块就会被打包。如果传入的是一个函数的话，其返回值必须是布尔类型来指明这个模块是否应该被打包进公共模块。而`chunks`则会指定一个字符串数组，如果设置了该参数，则打包的时候只会从其中指定的模块中提取公共子模块。

下面通过几个实例来说明这个插件是如何工作的。

假设有两个模块`chunk1.js`和`chunk2.js`以及两个项目文件`a.js`和`b.js`，结构大致如下：

```javascript
// a.js
require('./chunk1');
require('./chunk2');
require('jquery');
// b.js
require('./chunk1');
require('./chunk2');
require('vue');
// webpack配置如下
module.exports = {
  entry: {
    main: './main.js', 
    main1: './main1.js',        
    jquery:["jquery"],     
    vue:["vue"]   
  },    
  output: {      
    path: __dirname + '/dist',   
    filename: '[name].js' 
  },   
  plugins: [  
    new CommonsChunkPlugin({   
      name: ["common","jquery","vue","load"],   
      minChunks:2 
    })   
  ] };
```

最终的打包结果是：`jquery`被打包到`jquery.js`，`vue`被打包到`vue.js`，`common.js`打包的是公共模块(chunk1和chunk2)。使用该插件打包时，会将满足`minChunks`的模块打包到`name`数组的第一个块里，然后数组后面的依次打包，首先从`entry`中找，如果没有则产生一个空块。`name`数组中最后一个块打包的是webpack的runtime代码，在使用的时候必须先加载该块。

现在看一看`vue-cli`对于该插件的配置文件：

```javascript
new webpack.optimize.CommonsChunkPlugin({
  name: 'vendor',
  minChunks: function (module, count) {
    // 将node_modules中的依赖模块全部提取到vendor文件中
    return (
      module.resource &&
      /\.js$/.test(module.resource) &&
      module.resource.indexOf(
        path.join(__dirname, '../node_modules')
      ) === 0
    )
  }
}),
// webpack在使用CommonsChunkPlugin时会生成一段runtime代码，并且打包进vendor中。
// 这样即使不改变vendor代码，每次打包时runtime会变化导致vendor的hash变化，这里
// 把独立的runtime代码抽离出来来解决这个问题
new webpack.optimize.CommonsChunkPlugin({
  name: 'manifest',
  chunks: ['vendor']
}),
```

## 移除不必要的文件

在项目中我通过方法一定位到几处体积占用较大的库，其中一个便是`moment.js`这个日期处理库。对于一个日期处理的功能，为何这个库会占用如此大的体积，仔细查看发现当引用这个库的时候，所有的`locale`文件都被引入，而这些文件甚至在整个库的体积中占了大部分，因此当webpack打包时移除这部分内容会让打包文件的体积有所减小。

webpack自带的两个库可以实现这个功能：

* IgnorePlugin
* ContextReplacementPlugin

`IgnorePlugin`的使用方法如下：

```javascript
// 插件配置
plugins: [
  // 忽略moment.js中所有的locale文件
  new webpack.IgnorePlugin(/^\.\/locale$/, /moment$/),
],
// 使用方式
const moment = require('moment');
// 引入zh-cn locale文件
require('moment/locale/zh-cn');
moment.locale('zh-cn');
```

`ContextReplacementPlugin`的使用方法如下：

```javascript
// 插件配置
plugins: [
  // 只加载locale zh-cn文件
  new webpack.ContextReplacementPlugin(/moment[\/\\]locale$/, /zh-cn/),
],
// 使用方式
const moment = require('moment');
moment.locale('zh-cn');
```

通过以上两种方式，`moment.js`的体积大致能缩减为原来的四分之一。

## 模块化引入

在项目中我使用了`lodash`这个很常用的工具库，然而在代码定位的时候发现这个库也占了不少的体积。仔细想想，我们在使用这类工具库的时候往往只使用到了其中的很少的一部分功能，但却把整个库都引入了。因此这里也可以进一步优化，只引用需要的部分。

```javascript
import {chain, cloneDeep} from 'lodash';
// 可以改写为
import chain from 'lodash/chain';
import cloneDeep from 'lodash/cloneDeep';
```

这样就可以只打包我们需要的部分功能。

## 通过CDN引用

对于一些必要的库，但又无法对该库进行更好的体积优化的话，可以尝试通过外部引入的方式来减小打包文件的体积。采用该方法只需要在cdn站点找到需要引用的库的外部链接，以及对webpack进行简单配置即可。

```html
// 在html中添加script引用
<script src="https://cdnjs.cloudflare.com/ajax/libs/jquery/3.2.1/jquery.min.js"></script>
```

```javascript
// 这里externals的key指的是使用时需要require的包名，value指的是该库通过script引入后在全局注册的变量名
externals: {
  jquery: 'jQuery'
}
// 使用方法
require('jquery')
```

## 通过`DLLPlugin` 和 `DLLReferencePlugin` 拆分文件

如果项目过大，打包的时间会相当的长，如果频繁更新上线则会不断对代码进行编译打包，浪费很多时间。这时我们便可以将那些不常更新的框架和库(如vue.js等)进行单独的编译打包，这样每次开发上线就只需要对我们的开发文件进行编译打包，这样可以极大地省去不必要的打包时间。而这种方法需要`DLLPlugin`和`DLLReferencePlugin`两个插件的配合来完成。

### DLLPlugin

在使用这个插件时，我们需要单独创建一个配置文件，这里命名为`webpack.dll.config.js`，配置如下：

```javascript
module.exports = {
  entry: {
    lib: ['vue', 'vuex', 'vue-resource', 'vue-router']
  },
  output: {
    path: path.resolve(__dirname, '../dist', 'dll'),
    filename: '[name].js',
    publicPath: process.env.NODE_ENV === 'production'
      ? config.build.assetsPublicPath
      : config.dev.assetsPublicPath,
    library: '[name]_library'
  },
  plugins: [
    new webpack.DefinePlugin({
      'process.env': '"production"'
    }),
    /**
     * path: manifest.json输出文件路径
     * name: dll对象名，跟output.library保持一致
     */ 
    new webpack.DllPlugin({
      context: __dirname,
      path: path.resolve(__dirname, '../dist/dll', 'lib.manifest.json'),
      name: '[name]_library'
    })
  ]
}
```

这里要注意几点：

1. `entry`中写明所有要单独打包的模块
2. `output`的`library`属性可以将dll包暴露出来
3. `DLLPlugin`的配置中，`path`指明`manifest.json`文件的生成路径，`name`暴露出dll的函数名

运行该配置文件便可生成打包文件和`manifest.json`文件。

### DLLReferencePlugin

对于该插件的配置，不需要像上面一样单独写配置文件，只需要在生产配置文件中添加如下代码：

```javascript
new webpack.DllReferencePlugin({       
  context: __dirname,                  // 同dll配置的路径保持一致
  manifest: require('../dist/dll/lib.manifest.json') // manifest的位置
}),
```

然后运行webpack，发现打包的速度得到了极大地提升，也不用每次更新代码的时候重复编译打包这些依赖库了。

## 其他

对于webpack的打包优化我大致就总结了上面的一些方法，而为了让页面更快的加载，有更好的用户体验，我们并不只是从打包上优化，也可以有其他方面的优化，这里我也简单提一下我使用过的方法。

### 开启Gzip压缩

开启gzip压缩可以减少HTTP传输的数据量和时间，从而减少客户端请求的响应时间，由于降低了请求时间，页面的加载速度也会得到提升，会有更快的渲染速度，极大地改善了用户体验。由于现在基本上所有的主流浏览器都支持Gzip的压缩方式，只需要对服务器进行相关设置即可，这里就不具体讲如何配置服务器。

### 压缩混淆代码

我们平常也会对代码进行压缩混淆，可以通过`UglifyJS`等工具来对js代码进行压缩，同时可以去掉不必要的空格、注释、console信息等，也可以有效的减小代码体积。

## 总结

本文到这里就结束了，主要是对webpack的打包优化部分做了些讲解，当然能力和时间有限，只研究了部分方法，可能会有其他更多的优化方法，无论是从编译打包的体积还是速度上都能有更好的优化。接触了一段时间的webpack发现作为一个打包工具实在是过于复杂，无论从开始的官方文档还是到新的高级特性，都很难去完全掌握，还得需要自己不断去实践去深入研究才行。

