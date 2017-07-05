---
title: 记一次 webpack.optimize.UglifyJsPlugin 引起的 css vendor prefixes 被删除
date: 2017-07-05 15:14:09
tags: pack
---

## 事情起因

最近接手了一个 PC 端的项目，需要在原来的基础上添加上一些功能。项目的构建是用 webpack@1.x 进行构建。

原先构建的人为了兼容到 IE9， 构建的时候添加了 `postcss + autoprefixer` ，为 IE 添加 `-ms-` 前缀。

在开发调试的时候，测试兼容到 IE9。但是当部署生产环境代码给测试同学测试的时候，测试时 IE9 上有部分样式错乱问题，查看发现 `-ms-` 的 prefixer 消失。


### 不说废话上代码

下面是针对项目进行的模拟：

```
├── package.json
├── src
│   ├── index.css
│   └── index.js
└── webpack.config.js
```

依赖如下：（请注意版本）

``` json
{
  "devDependencies": {
    "autoprefixer": "^7.1.1",
    "css-loader": "^0.25.0",
    "postcss-loader": "^1.3.3",
    "style-loader": "^0.13.1",
    "webpack": "^1.13.3"
  }
}
```

src/index.css
``` css
.test-prefixer {
    transform: translate(-50%, -50%);
}
```

src/index.js
``` javascript
require('./index.css');
```

webapck.config.js
``` javascript
const path = require('path');
const webpack = require('webpack');

module.exports = {
    entry: './src/index',
    output: {
        path: path.resolve(__dirname, 'dist'),
        filename: '[name].js'
    },
    module: {
        loaders: [
            {
                test: /\.css$/,
                loader: 'style-loader!css-loader!postcss-loader'
            }
        ]
    },
    postcss: () => {
        return [
            require('autoprefixer')({
                browsers: ['IE>=9']
            })
        ];
    },
    plugins: (() => {
        let plugins = [];
        if (process.env.NODE_ENV === 'production') {
            plugins.push(new webpack.optimize.UglifyJsPlugin({
                compress: {warnings: false}
            }));
        }
        return plugins;
    })()
};
```

让我们查看一下非生产环境下的构建结果：

``` javascript
// ...
/***/ (function(module, exports, __webpack_require__) {

	exports = module.exports = __webpack_require__(3)();
	// imports


	// module
	exports.push([module.id, ".test-prefixer {\n    -ms-transform: translate(-50%, -50%);\n        transform: translate(-50%, -50%);\n}\n", ""]);

	// exports
// ...
```

再看一下生产环境下构建的结果：

``` javascript
// ....
t=e.exports=n(3)(),t.push([e.id,".test-prefixer{transform:translate(-50%,-50%)}",""])}
// ...
```

**`-ms-` 的 prefixer 消失了。。。**


## 排查

从 webpack.config.js 上看，两者唯一的区别是有没有添加 `webpack.optimize.UglifyJsPlugin` 进行 js 代码压缩。

**UglifyJs 压缩 JS 代码还能把 css 的 prefixer 给干了？？？？？？**

查了下 webpack 关于 `webpack.optimize.UglifyJsPlugin` 的 ISSUE, 找到最相似匹配的

[Using plugin "webpack.optimize.UglifyJsPlugin" led to remove the style "display: -webkit-flex;" ](https://github.com/webpack/webpack/issues/2543)

但是问题依然没有解决，`UglifyJs` 怎么可能去删除 js 的字符串中特定的字符！

### 从文档和源码开始

只能再看文档是不是忽略了什么 [webpack.optimize.UglifyJsPlugin](http://webpack.github.io/docs/list-of-plugins.html#uglifyjsplugin)

> Minimize all JavaScript output of chunks. `Loaders are switched into minimizing mode.` You can pass an object containing UglifyJS options.

请重点注意这句话： `Loaders are switched into minimizing mode.` **所有 loader 都会进入压缩模式**。

再来看看[源码](https://github.com/webpack/webpack/blob/v1.13.3/lib/optimize/UglifyJsPlugin.js#L143)：

``` javascript
// ...
compilation.plugin("normal-module-loader", function(context) {
    context.minimize = true;
});
```

为所有的 loader 添加了 `minimize` 配置。

### 回头看 webpack 配置

从上面 webpack 的配置上看，关于样式，只用了三个加载器， `style-loader`, `css-loader`, `postcss-loader`

对于  `style-loader` 和 `postcss-loader`， 都没有 `minimize` 的可选的配置项，那么问题只能出在 `css-loader` 上了。

那么我们再进行一次测试：

修改 css 的 loader 规则如下，强制在非生产环境压缩 css ，如下：

``` javascript
{
    test: /\.css$/,
    loader: 'style-loader!css-loader?minimize=true!postcss-loader'
}
```

查看结果如下：

``` javascript
// ...
/***/ (function(module, exports, __webpack_require__) {

	exports = module.exports = __webpack_require__(3)();
	// imports


	// module
	exports.push([module.id, ".test-prefixer{transform:translate(-50%,-50%)}", ""]);

	// exports
// ...
```

果然，问题是出在 `css-loader` 上，再针对 css-loader 进行处理就好了。查看[该版本 css-loader 的文档](https://github.com/webpack-contrib/css-loader/tree/v0.25.0#minification)：

> By default the css-loader minimizes the css if specified by the module system.

> `In some cases the minification is destructive to the css`, so you can provide some options to it. cssnano is used for minification and you find a list of options here. Just provide them as query parameter: i. e. require("css-loader?-autoprefixer") to disable removing of deprecated vendor prefixes.

对应的 ISSUE ： [Remove Autoprefixer from cssnano](https://github.com/webpack-contrib/css-loader/issues/281)

## 解决方案

又这么几种方式解决：

+ 1、告诉 css-loader 在进行压缩的时候，强制不删除 autoprefixer
    - `style-loader!css-loader?-autoprefixer!postcss-loader`
+ 2、强制不使用 css-loader 的压缩功能
  - `style-loader!css-loader?-minimize!postcss-loader`
  - 再手动指明用 [cssnano](https://github.com/ben-eb/cssnano) 对 css 进行压缩（配置不要压缩 autoprefixer 项）
+ 3、升级 css-loader： 这个问题在更高版本的 css-loader 中已经被修复，更新 css-loader 大于等于 `V0.26.0` 问题就可以解决， 不需要修改 webpack.config.js。
  - 可以对比下 css-loader 在使用 cssnano 压缩 css 代码的配置上又什么不同 [v0.25.0](https://github.com/webpack-contrib/css-loader/blob/v0.25.0/lib/processCss.js#L182) VS [v0.26.0](https://github.com/webpack-contrib/css-loader/blob/v0.26.0/lib/processCss.js#L182)
  - 对应的修复说明： [Disable Autoprefixer in cssnano](https://github.com/webpack-contrib/css-loader/pull/361)

其实上面几种方法，最终都是告诉 cssnano 在压缩时，不要去删除 autoprefixer 的内容。推荐使用方法3，其次是方法1，最后才是方法2。


## 总结

在 overstackflow 和 webpack@1.x 的 ISSUE 中出现的 webpack.optimize.UglifyJsPlugin 引起的 css prefixer 消失的问题根源还是 webpack.optimize.UglifyJsPlugin 会默认让所有 loader 进入压缩模式，然后触发了v0.26.0 之前版本 css-loader 调用 cssnano 压缩 css 时删除了 prefixer。
