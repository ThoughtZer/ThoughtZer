---
title: webpack
index_img: /img/webpack/webpack.png
tags: JavaScript三方库
date: 2019-08-16 20:20:30
---

webpack 是一个现代 JavaScript 应用程序的静态模块打包工具。当 webpack 处理应用程序时，它会在内部构建一个 依赖图(dependency graph)，此依赖图会映射项目所需的每个模块，并生成一个或多个 bundle。
<!-- more -->

(PS: 本文使用的是 v4 版本的 webpack)

### 核心概念

先从核心的概念说起吧~~

- **入口(entry)** 是入口文件的配置，可以是一个文件，多个文件(数组)或者对象，作为 webpack 构建依赖图的开始
- **输出(output)** 是 webpack 打包出 bundle 的出口，包含在哪里输出、输出什么文件名
- **loader** 是提高 webpack 转化能力的高级接口(webpack 只识别Js和Json)，能帮助 webpack 识别更多的文件，并转换成有效的模块
- **插件(plugin)** 是为了解决 loader 无法实现的其他事，比如打包优化，资源管理，注入环境变量...主要能开是监听 webpack 构建过程中的钩子函数，在特定的时间触发特性的事件
- **模式(mode)** 是指 webpack 当前运行的环境，可以对不同环境下执行流程进行优化变更。(可选值: development, production 或 none)
- **Chunk** 块，就像是堆积木的一个个单元(模块)，也可以是多个单元组成的一个单元(模块)。

### 流程

了解了 webpack 的核心概念，那么再看下 webpack 的构建流程，之后会对流程中一些细节进行说明

- 初始化参数：从配置文件```(webpack.config.js)```和 ```Shell``` 语句中读取与合并参数，得出最终的参数
- 开始编译：用上一步得到的参数初始化 ```webpack Compiler``` 对象，加载所有配置的插件，执行对象的 run 方法开始执行编译
- 确定入口：根据配置中的 ```entry``` 找出所有的入口文件
- 编译模块：从入口文件出发，调用所有配置的 ```Loader``` 对模块进行翻译，再找出该模块依赖的模块，再递归本步骤直到所有入口依赖的文件都经过了本步骤的处理
- 完成模块编译：在经过第4步使用 ```Loader``` 翻译完所有模块后，得到了每个模块被翻译后的最终内容以及它们之间的依赖关系(dependency graph)
- 输出资源：根据入口和模块之间的依赖关系，组装成一个个包含多个模块的 ```Chunk```，再把每个 ```Chunk``` 转换成一个单独的文件加入到输出列表
- 输出完成：在确定好输出内容后，根据 ```output``` 配置确定输出的路径和文件名，把文件内容写入到文件系统

一个简单的 ```webpack.config.js``` 配置内容:

```js
const path = require('path');

module.exports = {
    // entry:'./src/js/main.js',
    // entry:['./src/js/main.js','./src/js/a.js'],
    // entry 是入口文件的配置，可以是一个文件，多个文件(数组)或者对象
    entry:{
        main:'./src/js/main.js',
        a:'./src/js/a.js'
    },
    //output输出文件的存放位置和名称配置
    output:{
        path: path.resolve(__dirname, './dist/js'),
        // filename:'bundle.js'
        filename:"[name].bundle.js"
    },
    module:{
        rules: [
            {
                test: /\.js?$/,
                loader: 'babel-loader',
                exclude: /node_modules/, // 如果不去除node_modules则会导致打包编译速度过慢
            },
            // something...
        ]
    },
    plugins: [
        new CleanWebpackPlugin(), // 清除上一次打包结果(高版本不用传递参数)
        new HtmlWebpackPlugin({
            template: path.resolve(__dirname, './template/index.html'), // 依据模板生成html
        })
    ],
    resolve: {
        extensions: ['.js', '.jsx', '.vue'], // 引入模块的时候扩展
        mainFiles: ['index', 'main'], // 引入一个文件夹的时候默认寻找的文件开头名
        alias: {
            '@': path.resolve(__dirname, '../src'),
            'common': path.resolve(__dirname, '../src/common')
        }
    }
};
```

> 注意: 
> 在webpack配置文件中的路径，一般属于自己的文件系统的路径前面要加上```./```,否则 webpack 可能会去node_modules文件夹找依赖而报错

### 特别的能力

上面的一个 webpack 配置是一个很基础的打包功能，但是 webpack 的功能远不至此。一些另外的强大功能，也会让我们的开发变得更有效率

#### Babel 转译

[babel](https://babeljs.io/) 一个 JavaScript 编译器。配合 webpack 就能让我们使用最先进的 Js 语法进行编码，且产出浏览器可识别运行的 Js 代码。

下载

```sh
npm install --save-dev babel-loader @babel/core
npm install @babel/preset-env --save-dev
```

使用

```js
// webpack.config.js
rules: [
    {
        test: /\.js$/,
        exclude: /node_modules/,
        loader: "babel-loader"
    }
]
// .babelrc
{
    "presets": ["@babel/preset-env"]
}
```

如果我们没有配置一些规则，Babel 默认只转换新的 JavaScript 句法，而不转换新的 API，比如 Iterator、Generator、Set、Maps、Proxy、Reflect、Symbol、Promise 等全局对象，以及一些定义在全局对象上的方法都不会转码。

此时需要我们去引入另外的一些方法，去让 babel 支持以上所不能转换的操作。

##### babel-polyfill & babel-runtime

babel-polyfill 会对当前的运行环境中没有实现的方法进行兼容垫片处理，但是他用的方法是全局变量，并且不管你有没有使用这个方法，都会为你处理，导致最后打包之后的结果文件体积增大。

使用

```js
npm install --save @babel/polyfill
// main.js  
import '@babel/polyfill';
```

在 Babel v7 版本之后，对 polyfill 进行了按需加载能力扩展。详见[官网](https://babeljs.io/docs/en/babel-preset-env#corejs)

```js
npm install --save core-js@3  // @babel/polyfill 在 Babel v7.4 之后废弃，使用 core-js 替代

// .babelrc
{
    "presets": [
        ["@babel/preset-env", {
            "useBuiltIns": "usage", // "usage" | "entry" | false defaults to false.
            "corejs": "3",
            "modules": false,
            "targets": {
                "browsers": ["> 1%", "last 2 versions", "not ie <= 8"]
            },
        }]
    ]
}
```

babel-runtime 是采用其它方式实现新版 JavaScript 方法或者 API 能力的 Babel 扩展。它不会和 babel-polyfill 一样在全局作用域挂载。

它包含两个 runtime 的扩展:

- @babel/runtime
- @babel/plugin-transform-runtime

关于有两个 runtime 扩展原因是: runtime 扩展是在代码里面引入的各个 helper 函数，意味着代码里面可能有很多重复的垫片代码。plugin-transform-runtime 是把我们的代码里面需要兼容的那部分代码统一更换为一个名字，然后只实现这个被重写的需要兼容的垫片代码。**后者会比前者打包产出相对少的代码**。

```js
npm install --save-dev @babel/plugin-transform-runtime
npm install --save @babel/runtime
npm install --save @babel/runtime-corejs2
// .babelrc
{
    "plugins": [
    [
      "@babel/plugin-transform-runtime",
      {
        "corejs": 2,
        "helpers": true,
        "regenerator": true,
        "useESModules": false
      }
    ]
  ]
}
```

#### devtool

[devtool](https://webpack.js.org/configuration/devtool/#root) 是对 webpack 打包生成文件中是否包含 source map 的关键配置。有关 source map 简单理解就是 webpack 打包压缩代码之后和源码之间的一种映射关系。通过 source map 能快速的在浏览器的 debug 模式中找到存在问题的代码，从而进一步的调试。在官方文档中 devtool 支持 **7** 种模式，每一种模式的对比都有详尽的说明，不再进行阐述。

在这里只关心两个常用的模式选项:

```cheap-module-eval-source-map``` （开发环境推荐）

```cheap-module-source-map``` （生产环境推荐）

补充说明: 有的小伙伴也可以认为生产环境不能产出source-map，避免代码泄露，个人觉得每一种说法都有自己的道理，没有对与错，看各自项目的需要，适合自己的才是最好的。

对于以上两个模式的选择原因进行简要说明:

- 使用 cheap 模式可以大幅提高 soure map 生成的效率。大部分情况我们调试并不关心列信息，而且就算 source map 没有列，有些浏览器引擎（例如 v8） 也会给出列信息。

- 使用 eval 方式可大幅提高持续构建效率。官方文档提供的速度对比表格可以看到 eval 模式的编译速度很快。虽然不是很全面，但不影响使用。

- 使用 module 会去生成 node_module 里面的三方模块的 source map ，不仅仅关注业务代码。

- 使用 eval-source-map 模式可以减少网络请求。这种模式开启 DataUrl 本身包含完整 source map 信息，并不需要像 sourceURL 那样，浏览器需要发送一个完整请求去获取 source map 文件，这会略微提高点效率。而生产环境中则不宜用 eval，这样会让文件变得极大。

#### 自动刷新 -> 热更新

在前端还没有进入构建工具的时代里，前端开发者对于开发过程中代码的调试都是在更改完成代码之后，手动刷新浏览器查看运行效果是否达到预期。或者使用 ```live-server``` 之类的模拟服务来达到自动刷新浏览器的目的。之后前端进入构建工具时代，webpack 带领我们走向了一个更加智能化的热更新道路。

webpack 支持两个不同的自动刷新功能:

- --watch 监听到文件的变化就自动刷新浏览器。与 ```live-server``` 这类工具大致功能相同
- devServer 依赖于 webpack 强大的插件扩展系统，不仅仅能做到自动刷新，甚至于不刷新浏览器，只更新相关模块的变化。**最重要的一点是能够解决前后端分离之后的开发过程中接口跨域问题**

一个简单的 devServer 配置内容:

```js
devServer: {
    contentBase: './dist', // 在哪里启动服务
    open: true, // 自动打开浏览器
    proxy: { // 接口转发
        '/api': {
            target: '', // 目标地址
            changeOrigin: true, // 是否更改请求的 host origin
            secure: false, // https or not
            pathRewrite: {
                '/api': '', // 是否转发之后把前缀更改
            },
            headers: { // 转发所携带的headers信息
                host: '',
                cookie: ''
            },
            historyApiFallback: true, // 避免 SPA 应用刷新之后出现 404 找不到文件，因为 SPA 只有一个 html 文件。
            // 此参数有很多其他选项，可参考官网 https://webpack.js.org/configuration/dev-server/#devserverhistoryapifallback
        },
    },
    port: 9999, // 端口
    hot: true. // 是否热更新
    hotOnly: true, // 热更新出错是否继续
}
plugins: [
    new webpack.HotModuleReplacementPlugin(), // 配合热更新插件
]
```

关于热更新需要说明的是:
在 webpack 中，原始的热更新是需要在业务代码中去对 module.hot 进行判断只有再次对需要更新的文件进行移除、再引入等操作，才能达到热更新效果的。目前各种 webpack 的 loader 以及 babel 的 preset 已经对相关内容进行了处理，所以开发者在编写业务代码的时候，就无需再次进行编写这部分热更新逻辑。

#### eslint 代码风格检测

eslint 能在一个项目中保证在代码风格的统一，对于一个团队来说这个是很重要的。

```js
npm install eslint -D

// webpack.config.js
{
    test: /\.js$/,
    use: [
        'babel-loader',
        {
            loader: 'eslint-loader',
            options: {
                fix: true, // 是否每次检测自动修复问题
            },
            force: 'pre', // 注意eslint-loader是配合babel-loader使用，但是一般都是在babel转译之前进行代码风格检测
        },
    ]
}

devServer: {
    overlay: true, // 对于检测出来的错误信息是否放到浏览器页面中展示
}
```

#### 环境(宏、全局)变量

前端工程化之后带来的不仅仅是模块化、组件化、自动编译等等这些便利。日常软件系统的开发大多数要经历**开发**、**测试**、**预发布**、**线上**多个环境的部署调试最终到达线上生产环境，环境的多变对于代码层面的分割就有一定的要求，可能就存在不同环境下需要打包不同的代码逻辑或者不同的依赖，所以环境变量也就可能需要被部分业务代码所依赖。

- mode

在 webpack 中提供了一个 [mode](https://webpack.js.org/configuration/mode) 参数

> Possible values for mode are: none, development or production(default). 支持 none | development | production(默认) 三个选项

这个参数在 webpack 打包开始的时候被注入，内部的 webpack 配置可以根据这个参数做不同的打包处理。但是这种方式有一定的局限性，不能在Js代码中获取相对应 mode 参数。

- DefinePlugin —— webpack 一个内置插件，可以用来实现类似于宏替换的功能。有时候我们不仅仅想在打包的过程中对一些变量进行判断，在业务里面也想使用，那么 DefinePlugin 就发挥作用了

```js
plugins: [
  new webpack.DefinePlugin({
    isDev: env.mode === 'development'
  }),
],
```

之后在业务代码里面可以这样:

```js
if (isDev) {
    /* something */
}
```

但是要注意的是 DefinePlugin插件并不是定义了一个叫做isDev的变量，而是将代码中的 ```isDev``` 用编译时 ```env.mode === 'development'```表达式的值替换。（**DefinePlugin做的是代码中的宏替换，不要把它当做定义变量来使用**）所以编译之后的代码:

```js
// 如果 env.mode === 'development'
if (true) {
    /* something */
}
// 如果 env.mode !== 'development'
if (false) {
    /* something */
}
// 或者被直接删掉相关逻辑代码
```

这样的例子比较简单，有时候我们需要在代码中根据不同的版本请求不同的接口，一般来说在项目中的 package.json 中有 version 字段标明目前是第几个版本。就需要把这个版本当做一个宏注入到业务代码中进行使用:

```js
// webpack.config.js
const version = require('./package.json').version;
plugins: [
  new webpack.DefinePlugin({
    '__VERSION__': version,
  }),
],

// 业务代码模块
const version = __VERSION__;
export {version};
```

当然我们可以将 package.json 直接 import 进来然后将 version 属性导出，但是这么做会把整个 package.json 中的内容全都打包进模块，如果我们只是使用其中的 version 属性，那么打包一整个 package.json 文件也没必要。

- cross-env

[cross-env](https://github.com/kentcdodds/cross-env) 是一个跨平台的设置环境变量的脚本 (Cross platform setting of environment scripts)，跨平台指的是操作系统层面的~

```js
// package.json
{
  "scripts": {
    "build": "cross-env NODE_ENV=production webpack --config build/webpack.config.js"
  }
}
// webpack.config.js
if (process.env.NODE_ENV === production) {
    /* something */
}
// 配合 DefinePlugin
plugins: [
  new webpack.DefinePlugin({
    'process.env': {
      NODE_ENV: JSON.stringify(process.env.NODE_ENV)
    }
  }),
],
```

#### 代码优化

##### Tree Shaking [摇树](https://webpack.js.org/guides/tree-shaking/#root)

简单来说 Tree Shaking 的能力就是把引入模块中没有用到的东西都删除掉，从而减小打包之后文件体积

> 目前只支持 ESModule 静态引入方式(import), 且 webpack 的 development 模式下 Tree Shaking 没有开启

- 开发环境 Tree Shaking

由于开发环境默认是不启动状态，所以需要手动开启 Tree Shaking

```js
// webpack.config.js
optimization: {
    usedExports: true
}
```

当然仅仅这样来做是不够的，因为 webpack 并不知道摇树之后会带来什么样的影响，所以 v4 版本的 webpack 提供了一个 ```sideEffects``` 标识来标记哪些文件是摇树会造成影响的。如果都没有影响则直接可以设为```false```，那么 webpack 就会放心大胆的处理文件了

```js
// package.json
{
  "name": "project-name",
  "sideEffects": false,
  /*
  如果有多个文件需要一一进行匹配配置
  "sideEffects": [
      "./src/some-side-effectful-file.js",
      "*.css"
  ]
  */
}
```

- 生产环境 自动开启, 但是 package.json 中 useEffects 必须要

##### Code splitting [代码分割](https://webpack.js.org/guides/code-splitting/#root)

代码分割的作用:
拆分出不经常更改的三方库或者独立逻辑，防止每次业务逻辑更改重新打包，用户多次请求相同的内容浪费资源

- splitChunks 插件拆分

贴一个官网的简单配置，分析一下:

```js
// webpack.config.js
optimization: {
    splitChunks: {
        chunks: 'all', // 同步异步引入都打包
        <!--以下配置可以自己选择性配置-->
        minSize: 30000, // 至少大于30kb
        maxSize: 0, // 拆出的宝最大的大小配置，如果大于此值，会被再次拆分(一般不需要)
        minChunks: 1, // 在打包生成之后的chunk里面被引入一次就算
        maxAsyncRequests: 5, // 打包生成的文件最多是5个，多余5个不分
        maxInitialRequests: 3, // 入口文件的拆分最多3个
        automaticNameDelimiter: '~', // 拆分出的包与主包之间名称连接符
        name: true, // 自定义名称是否生效
        cacheGroups: {
            // 来自node_modelues中的包打一个
            vendors: {
                test: /[\\/]node_modules[\\/]/,
                priority: -10,
                filename: 'vendors.js',
            },
            // 不满足以上条件的包打一个
            default: {
                minChunks: 2,
                priority: -20,
                reuseExistingChunk: true, // 如果已经被打包过，则不在重复进行拆包
            }
        }
    }
}
```

- 动态引入拆分 (import语法)

动态引入的意思就是在需要的时候引入(import导入语法返回一个 promise )，比如:

```js
if (条件为真) {
    // 需要引入
    import('third-party-module / other-module').then((module) => {
        // something
    })
} else {
    // 不需要引入
}
```

在 webpack 中对这种方式的语法还进行了一种 Magic Comment 语法增强:

```js
import(/* webpackChunkName: "lodash" */ 'lodash').then(() => { /* something */ })
```

这种语法会把 import 引入的模块拆分到一个 webpackChunkName.bundle.js。并且多个模块可以配置相同的 webpackChunkName，从而更好的拆分代码。需要注意的是这种import 动态引入的语法是需要 babel 转译的支持的

```js
npm install @babel/plugin-syntax-dynamic-import -D
```

##### 预获取/预加载(prefetch/preload)

webpack v4.6 之后添加了预获取和预加载的功能。在声明 import 时，使用 prefetch / preload 指令的 Magic Comment，可以让 webpack 输出 "resource hint(资源提示)"，来告知浏览器：

- prefetch(预获取)：将来某些导航下可能需要的资源
- preload(预加载)：当前导航下可能需要资源

两者的具体的不同之处可以查看[官方文档](https://webpack.js.org/guides/code-splitting/#prefetchingpreloading-modules)。但是需要谨记的是一般会经常使用 prefetch 在浏览器闲置的情况下加载将来可能需要的资源！不正确地使用 preload 会有损性能。

##### CSS 的拆分与压缩

之前所说的代码分割都是主要针对于 JavaScript 代码的操作，但是随着项目的进行文件可能越来越多，复杂度也随之上升，CSS也会逐渐增大，对 CSS 的拆分压缩处理也就不可避免了，当然为了提升开发效率，**推荐在生产环境打包情况下进行此项操作**。

- 拆分(mini-css-extract-plugin)

```js
npm install mini-css-extract-plugin -D
// webpack.config.js
const MiniCssExtractPlugin = require('mini-css-extract-plugin');
plugins: [
    new MiniCssExtractPlugin({
        filename: 'css/[name].[hash:8].css',
        chunkFilename: 'css/[id].[hash:8].css',
    }),
],
module: {
    rules: [
        {
            test: /\.(css)/,
            use: [
                {
                    loader: MiniCssExtractPlugin.loader,
                },
                'css-loader',
                'postcss-loader'
            ],
        }
    ]
},
```

- 压缩 (optimize-css-assets-webpack-plugin)

```js
npm install optimize-css-assets-webpack-plugin -D
// webpack.config.js
const OptimizeCSSAssetsPlugin = require('optimize-css-assets-webpack-plugin');

optimization: {
    minimizer: [
        new OptimizeCSSAssetsPlugin({})
    ],
},
```

### 提高打包速度

用过 webpack (包含```vue-cli```或者```CRA```)的小伙伴可能都会发现当项目的依赖越来越多的时候，打包的过程就非常的耗时，这也是 webpack 最令人诟病的一点吧。好在 webpack 提供了一些解决办法:

#### DLLPlugin + DllReferencePlugin

DLLPlugin 和 DllReferencePlugin 是 webpack 的内置插件。具体使用方式参考如下:

```js
// webpack.dll.js
const path = require('path');
const webpack = require('webpack');

module.exports = {
    mode: 'production',
    entry: {
        // 拆分三方库到dll
        vendors: [
            'lodash'
        ],
        vue: [
            'vue',
            'vue-router',
        ],
        iview: [
            'iview'
        ]
    },
    output: {
        filename: '[name].dll.js',
        path: path.resolve(__dirname, '../dll'),
        library: '[name]',
    },
    plugins: [
        // 生成dll映射关系manifest
        new webpack.DllPlugin({
            name: '[name]',
            path: path.resolve(__dirname, '../dll/[name].manifest.json')
        })
    ]
}
// webpack.config.js
const path = require('path');
const webpack = require('webpack');
// 以下2个依赖需要 npm install
const glob = require('glob');
const AddAssetHtmlWebpackPlugin = require('add-asset-html-webpack-plugin');

const assetHtml = [];
const dllReference = [];

glob.sync('./dll/**.dll.js').forEach(item => {
    const chunk = item.split('./dll/')[1].split('.dll.js')[0];
    assetHtml.push(
        new AddAssetHtmlWebpackPlugin({
            filepath: path.resolve(__dirname, `../dll/${chunk}.dll.js`)
        }),
    );
});

glob.sync('./dll/**.manifest.json').forEach(item => {
    const chunk = item.split('./dll/')[1].split('.manifest.json')[0];
    dllReference.push(
        new AddAssetHtmlWebpackPlugin({
            filepath: path.resolve(__dirname, `../dll/${chunk}.manifest.json`)
        }),
    );
});

plugins: [
    ...assetHtml,
    ...dllReference,
],
```

#### externals + CDN

externals 是为了防止将某些import的包(package)打包到 bundle 中，而是在运行时(runtime)再去从外部获取这些扩展依赖。
所以，可以将体积大的库分离出来:

```js
// webpack.config.js
externals: {
    'iview': 'iView',
    'v-charts': 'VCharts'
}

// 移除业务代码中 import 语句
// 在 模板 index.html 中添加 (ps: 以下cdn地址是不存在的，只是为了说明)
<script src="https://cdn.com/iview/lib/index.js"></script>
<script src="https://cdn.com/echarts/dist/echarts.min.js"></script>
<script src="https://cdn.com/v-charts/lib/index.min.js"></script>
```

#### 社区提供的 happypack 采用多进程打包，单个进程处理完成只有交回主进程的方式。详见[官方文档](https://github.com/amireh/happypack)。

除了以上说到的三个方式之外，还有一些需要注意的小点，在配置 webpack 的时候仔细检查即可，无需复杂的配置:

- 升级Node、webpack 版本(npm、yarn)

- 在尽可能少的模块中使用loader(includes/excludes)

- 尽可能少的使用plugin、确保plugin的可靠性

- resolve 字段的合理配置

- 注意区分多个环境打包配置的多样性

### 多页面

因为之前参加工作的时候有做过微信公众号相关的项目，对于微信这种可能需要依赖其庞大的用户群体进行流量积攒的项目来说，分享、调用微信API之类的功能是必不可少的，如果要是对于分享的页面存在于一个 SPA 页面中，那么对于用户来说不算是很友好的(开发者也不喜欢这样做，会有很多潜在的坑)。所以多页面就又有了价值的提现，对于 webpack 来说，多页面就是多个 entry 打包出多个相对应的 html。

但是随着业务的不断扩展，页面可能会不断的追加，所以一定要让入口的配置足够灵活，避免每次添加新页面还需要修改构建配置文件，所以需要约定一种格式——文件存放的格式。笔者有进行过一些尝试，[详见](https://github.com/ThoughtZer/vue-multipage-template)，如有问题，请指正。

主要的 webpack 配置:

```js
// webpack.config.js
const path = require('path');
const glob = require('glob');
const HtmlWebpackPlugin = require('html-webpack-plugin');

let HTMLPlugins = [];
let Entries = {};

glob.sync('./src/pages/**/index.js').forEach(item => {
  const chunk = item.split('./src/pages/')[1].split('/index.js')[0];
  Entries[chunk] = item;

  const filename = chunk + '.html';
  const htmlConf = {
    filename: filename,
    template: path.resolve(__dirname, `../src/template/template.html`),
    inject: 'body',
    hash: true,
    chunks: [chunk, 'vendor'],
  };
  HTMLPlugins.push(new HTMLWebpackPlugin(htmlConf));
});

module.exports = {
    entry: Entries,
    plugins: [
        ...,
        ...HTMLPlugins,
    ]
}
```

### 结尾

到这里基本上 webpack 的常用的一些配置已经足够日常开发使用，配合其社区的强大 Plugin 以及 Loader 系统，应该是能满足各种各样个性化的需求配置，webpack 是很强大的工程化工具，前端开发者还是需要多尝试体验其中的功能的，以上内容是作为以往的学习笔记记录，如有错误，请指正。
