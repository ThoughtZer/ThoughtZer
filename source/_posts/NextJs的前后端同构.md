---
title: NextJs的同构使用记录
index_img: /img/next/next.png
tags: 框架
date: 2019-10-20 20:19:30
---

Next.js 是一个轻量级的 React 服务端渲染应用框架。
<!-- more -->

### 渲染方式

服务端渲染指的是页面的渲染和生成由服务器来完成，并将渲染好的页面返回客户端进行展示，例如: ASP、JSP、Smarty。

客户端渲染指的是页面的生成和数据的渲染过程是在客户端（浏览器或APP）执行JavaScript代码完成展示。例如: Angular、React、Vue。

Web页面的渲染方式从之前的服务端渲染到后续的客户端渲染SPA应用，解耦了前后端开发，使得前端专注于用户UI层，提升用户体验，让后端专注于业务逻辑处理，分离成微服务等。然而SPA对于SEO的支持非常之不友好，对于需要SEO的网站，SPA就不是一个很好的选择。所幸的是 React、Vue 都开始支持服务端渲染(同构)。

#### 同构

首屏的渲染交给服务端渲染，提升渲染速度。首屏加载之后，进入页面内部的路由交给客户端控制，不再经过服务端，降低服务器压力。目前基于React的[NextJS](https://nextjs.frontendx.cn/)以及基于Vue的[NuxtJs](https://zh.nuxtjs.org/)都是采用这样的方式。（本文只介绍 NextJS ）

### NextJS

特性:

- 默认服务端渲染模式，以文件系统为基础的客户端路由

- 代码自动分隔使页面加载更快

- 以webpack的热替换为基础的开发环境

- 使用React的JSX和ES6的module，模块化和维护更方便

- 可以运行在Koa和其他Node.js的HTTP 服务器上

- 可以定制化专属的babel和webpack配置

- 预置 CSS-in-JS 方案

#### 构建项目

推荐使用官方(非fb)提供的脚手架工具 *create-next-app*，直接构建。当然可以手动配置 *create-react-app*构建出来的项目。[详见](https://nextjs.org/docs#setup)

结构目录:

  |- components/* // 组件文件夹
  |- pages/*      // 页面文件夹
  |- static/*     // 静态资源文件夹
  |- .gitignore
  |- package.json

#### 开发

NextJS是根据pages目录下的文件名来确定页面的路由(支持文件夹嵌套)，因此pages目录是只能放置页面代码，不能放置其他代码的。而static目录是项目的静态文件目录，可以放置图片资源，并且Next也帮助我们配置 ```/static``` 作为引入资源的根路径。(emmmmm，限制有点多，所以初始化的目录，需要增添点~)

  |- components/*
  |- lib/*          // 资源库文件夹(自写或者不能npm install的包)
  |- pages/*
  |- server/*       // 自定义服务端代码
  |- static/*
  |- store/*        // redux
  |- style/*        // 如果不需要官方提供的css-in-js方案，可自定义其他方案(css-module或者styled-components)，放置样式
  |- .gitignore
  |- package.json
  |- next.config.js // next主配置文件

配置好目录之后，就是和一般的react项目一样进行开发就完事了，推荐使用 [hooks](https://reactjs.org/docs/hooks-intro.html)

#### 一些坑

- css-in-js 不支持除官方已经使用的```style-jsx```之外的多种css-in-js方案合集

比如不支持同时使用```css-module```和```styled-components```，否则可能会在页面中使用Link前端跳转页面失败。当然如果不需要保存页面状态，可以直接使用 a 标签跳转，没有此问题。[详情可见](https://github.com/zeit/next-plugins/issues/282#issuecomment-523696006)

- 异步组件无法使用 ref 获取组件实例

因为NextJS是每个页面都可能在服务端渲染的，所以对于一些使用了浏览器对象(如: window、document等)的JavaScript库，不能直接引入。幸好的是NextJS提供了[```dynamic```](https://nextjs.org/docs#dynamic-import)引入，选择```No SSR```引入使用此类库的子组件，可以解决此类的问题。当然万事有利有弊，使用此类方式引入的子组件就不能被父组件通过 ref 获取到子组件的实例，从而不能在父组件调用子组件的任何方法，因为异步组件是一个```LoadableComponent```[详见 react 源码](https://github.com/facebook/react/blob/f6b8d31a76cbbcbbeb2f1d59074dfe72e0c82806/packages/react/src/ReactElement.js#L371-L384)

- 自定义node服务，如果使用```Koa-router```不支持使用，```router.get/post``` 此类方式编写接口，需使用中间件，传递```ctx```，进而判断```path```

```js
// api.js
const api = (server) => {
  server.use(async (ctx, next) => {
    const { path, method } = ctx;
    if (path.startWith('xxxx')) { // 满足接口的path条件
      // do something
    } else {
      await next();
    }
  })
}
// server.js
const server = new Koa();
api(server)
```

当然如果不想在同一个项目里面写接口和渲染的话，可以另起Api接口项目，使用node服务转发。主要通过```http-proxy-middleware```和```koa-connect```。需要注意的是```http-proxy-middleware```转发接口，如果是上传资源之类的接口，当接口持续时间过长没有返回接口，代理会被断开，导致接收不到响应。但是线上推荐使用[```nginx```](http://nginx.org/en/)。

```js
const Koa = require('koa');
const next = require('next');
const c2k = require('koa-connect');
const proxyMiddleware = require('http-proxy-middleware');

const dev = process.env.NODE_ENV !== 'production';
const app = next({ dev });
const handle = app.getRequestHandler();

const devProxy = {
  target: '转发接口地址',
  changeOrigin: true,
  secure: false,
  onError(err, req, res) {
    res.writeHead(500, {
      'Content-Type': 'text/plain',
    });
    res.end(
      'Something went wrong. And we are reporting a custom error message.',
    );
  },
};
app.prepare().then(() => {
  const server = new Koa();
  // 开发环境代理接口，线上推荐使用nginx
  if (dev && devProxy) {
    server.use(async (ctx, nextKoa) => {
      if (ctx.url.startsWith('/api')) { // 以api开头的请求全部转发
        // ctx.respond = false;
        await c2k(proxyMiddleware(devProxy))(ctx, next);
      } else {
        await nextKoa();
      }
    });
  }
  // 中间件
  server.use(async (ctx) => {
    // 传入的是nodeJS原生的req、res达到兼容更多的node框架
    const { req, res } = ctx;
    await handle(req, res);
    ctx.respond = false;
  });
  server.listen(8888, () => {
    console.log('koa server listen on port 8888'); // eslint-disable-line
  });
});
```

- redux无法使用模块化，只能通过对象属性方式获取redux内容

在一般的SPA页面中，使用```redux```，在项目模块增多的时候，会使用```combineReducers```组合拆分各个模块的redux数据。然后在组件中使用```react-redux```连接redux，获取数据。

```js
import { connect } from 'react-redux';

const Components = () => {
  // 组件
}
const mapStateToProps = (state) => {
  return {
    reduxProp: state.getIn(['模块名', '需要获取redux的该模块下的某一字段的key']),
  };
};
export default connect(mapStateToProps, null)(Components);
```

但是在NextJS中，即使我们使用了```combineReducers```组合拆分各个模块的redux数据，我们在组件内部使用```react-redux```连接redux，获取数据的时候，不能通过```state.getIn(['', ''])```的方式获取，因为state没有经过处理，只能通过```state['模块名']['需要获取redux的该模块下的某一字段的key']```的方式获取。

```js
const mapStateToProps = (state) => {
  return {
    reduxProp: state['模块名']['需要获取redux的该模块下的某一字段的key'],
  };
};
```

- static 文件夹存放的图片资源压缩问题

在开发SPA应用的时候，经常会使用 webpack 进行打包编译，NextJS也是。但是不同的是: NextJS规定了静态文件路由 static，打包编译过程中不会处理这些文件，且打包之后启动服务同样使用的 static 文件。所以在图片压缩这个上面， NextJS 可能无法做到像一般的SPA一样通过 loader 处理。笔者使用的方式是通过编写一个 node 压缩图片脚本，把图片输出到 static 目录使用。主要使用```imagemin```、```imagemin-jpegtran```、```imagemin-pngquant```三个来自于[```image-webpack-loader```](https://github.com/tcoopman/image-webpack-loader#readme)的依赖。

- getInitialProps 生命周期钩子

NextJS提供了一个很强大的生命周期钩子: ```getInitialProps```，会在服务端渲染和客户端渲染的时候都执行，但是只要服务端执行了客户端就不会执行。所以一般用来当做请求数据的钩子使用。

有些时候我们可能需要对一些数据随机的进行展示，两种可能采用的方式:

1. 在```getInitialProps```中返回数据到组件的 props 中，然后再随机取值，渲染
2. 在```getInitialProps```中返回随机去的数据到组件的 props 中，然后再渲染

需要注意的是一旦采用了函数式组件的编写模式，方式1是不可取的，因为一旦当前页面被当做首屏，进行服务端渲染，服务端是会执行```getInitialProps```以及组件函数，浏览器渲染的时候不会执行```getInitialProps```，但是同样会执行组件函数，导致前后端数据存在不一致的可能就会发生。因此推荐第二种方式，数据的处理统一放置于```getInitialProps```。

文章所用NextJS版本为9.1.1~
