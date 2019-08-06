---
title: axios 源码简析(一)
index_img: /img/axios/axios.png
tags: JavaScript三方库
date: 2019-08-04 19:12:21
---

Promise based HTTP client for the browser and node.js —— 基于 Promise 的 HTTP 库，可以用在浏览器和 NodeJs 中
<!-- more -->

### 特点(能力)

- 支持从浏览器中创建 XMLHttpRequests
- 支持从 NodeJs 创建 http 请求
- 支持 Promise API
- 拦截请求和响应
- 转换请求数据和响应数据，自动转换 JSON 数据
- 取消请求
- 客户端支持防御 XSRF

目前 axios 使用范围非常广泛，前端开发者一般使用它来做 AJAX 请求，完成前后端数据交互。所以本篇文章主要分析在浏览器端的 axios 部分实现逻辑，包括

- [多种调用方式的实现](#如何实现多种调用方式)
- [配置的多处传递的实现](#从-config-配置参数入手吧)
- [拦截器的实现](#拦截器)
- 转换请求数据和响应数据的实现
- 取消请求的实现
- 自动转换JSON以及XSRF的部分实现

#### 目录结构

![目录结构](/img/axios/目录结构.png)

#### 如何实现多种调用方式

经常使用 axios 的小伙伴都应该知道 axios 的一些使用方式， 比如:

```js
import axios from 'axios';

axios(config) // 直接传入配置
axios(url[, config]) // 传入url和配置
axios[method](url[, option]) // 直接调用请求方式方法，传入url和配置
axios[method](url[, data[, option]]) // 直接调用请求方式方法，传入data、url和配置
axios.request(option) // 调用 request 方法

const axiosInstance = axios.create(config)
// axiosInstance 也具有以上 axios 的能力

axios.all([axiosInstance1, axiosInstance2]).then(axios.spread(response1, response2))
// 调用 all 和传入 spread 回调
```

针对以上多种使用方法，axios 内部实际上是在对外暴露接口的文件 axios.js 中体现的

```js
function createInstance(defaultConfig) {
  var context = new Axios(defaultConfig);

  // instance指向了request方法，且上下文指向context，所以可以直接以 instance(option) 方式调用 
  // Axios.prototype.request 内对第一个参数的数据类型判断，使我们能够以 instance(url, option) 方式调用
  var instance = bind(Axios.prototype.request, context);

  // Copy axios.prototype to instance
  // 把Axios.prototype上的方法扩展到instance对象上，
  // 并指定上下文为context，这样执行Axios原型链上的方法时，this会指向context
  utils.extend(instance, Axios.prototype, context);

  // Copy context to instance
  // 把context对象上的自身属性和方法扩展到instance上
  // 注：因为extend内部使用的forEach方法对对象做for in 遍历时，只遍历对象本身的属性，而不会遍历原型链上的属性
  // 这样，instance 就有了  defaults、interceptors 属性。
  utils.extend(instance, context);
  return instance;
}

// Create the default instance to be exported 创建一个由默认配置生成的axios实例
var axios = createInstance(defaults);

// Factory for creating new instances 扩展axios.create工厂函数，内部也是 createInstance
axios.create = function create(instanceConfig) {
  return createInstance(mergeConfig(axios.defaults, instanceConfig));
};

// Expose all/spread
axios.all = function all(promises) {
  return Promise.all(promises);
};

axios.spread = function spread(callback) {
  return function wrap(arr) {
    return callback.apply(null, arr);
  };
};

module.exports = axios;
```

主要核心是 ```Axios.prototype.request```,各种请求方式的调用实现都是在 ```request``` 内部实现的，
简单看下 ```request``` 的逻辑

```js
Axios.prototype.request = function request(config) {
  // Allow for axios('example/url'[, config]) a la fetch API
  // 判断 config 参数是否是 字符串，如果是则认为第一个参数是 URL，第二个参数是真正的config
  if (typeof config === 'string') {
    config = arguments[1] || {};
    // 把 url 放置到 config 对象中，便于之后的 mergeConfig
    config.url = arguments[0];
  } else {
    // 如果 config 参数是否是 字符串，则整体都当做config
    config = config || {};
  }
  // 合并默认配置和传入的配置
  config = mergeConfig(this.defaults, config);
  // 设置请求方法
  config.method = config.method ? config.method.toLowerCase() : 'get';
  /*
    something... 此部分会在后续拦截器单独讲述
  */
};

// 在 Axios 原型上挂载 'delete', 'get', 'head', 'options' 且不传参的请求方法，实现内部也是 request
utils.forEach(['delete', 'get', 'head', 'options'], function forEachMethodNoData(method) {
  Axios.prototype[method] = function(url, config) {
    return this.request(utils.merge(config || {}, {
      method: method,
      url: url
    }));
  };
});

// 在 Axios 原型上挂载 'post', 'put', 'patch' 且传参的请求方法，实现内部同样也是 request
utils.forEach(['post', 'put', 'patch'], function forEachMethodWithData(method) {
  Axios.prototype[method] = function(url, data, config) {
    return this.request(utils.merge(config || {}, {
      method: method,
      url: url,
      data: data
    }));
  };
});
```

#### 从 config 配置参数入手吧

config 配置应该是贯穿了 axios 的"一生"，从入口的axios.js中的 ```createInstance``` 到真实的请求方法 ```Axios.prototype.request```，所以我们目前就跟着它走着吧~

axios 中的 config 主要分布在这几个地方:

- 默认配置 ```defaults.js```
- config.method 默认为 ```get```
- 调用 ```createInstance``` 方法创建 axios 实例，传入的config
- 直接或间接调用 ```request``` 方法，传入的 config

这四处之间的优先级关系如下:

![配置优先级](/img/axios/config.png)

具体体现在源码中:

```js
// axios.js
// 创建一个由默认配置生成的axios实例
var axios = createInstance(defaults);

// 扩展axios.create工厂函数，内部也是 createInstance
axios.create = function create(instanceConfig) {
  return createInstance(mergeConfig(axios.defaults, instanceConfig));
};

// Axios.js
// 合并默认配置和传入的配置
config = mergeConfig(this.defaults, config);
// 设置请求方法
config.method = config.method ? config.method.toLowerCase() : 'get';
```

#### 真实请求的 request

在之前介绍的 axios 使用方法和配置 config 流转时，已经引出了 ```Axios.prototype.request``` 方法，这是 Axios 的一个核心方法，下面就针对这个核心方法进行分析

```js
Axios.prototype.request = function request(config) {
  /*
    先是 mergeConfig ... 等，不再阐述
  */
  // Hook up interceptors middleware 创建拦截器链. dispatchRequest 是重中之重，后续重点
  var chain = [dispatchRequest, undefined];

  // push各个拦截器方法 注意：interceptor.fulfilled 或 interceptor.rejected 是可能为undefined
  this.interceptors.request.forEach(function unshiftRequestInterceptors(interceptor) {
    // 请求拦截器逆序 注意此处的 forEach 是自定义的拦截器的forEach方法
    chain.unshift(interceptor.fulfilled, interceptor.rejected);
  });

  this.interceptors.response.forEach(function pushResponseInterceptors(interceptor) {
    // 响应拦截器顺序 注意此处的 forEach 是自定义的拦截器的forEach方法
    chain.push(interceptor.fulfilled, interceptor.rejected);
  });

  // 初始化一个promise对象，状态为resolved，接收到的参数为已经处理合并过的config对象
  var promise = Promise.resolve(config);

  // 循环拦截器的链
  while (chain.length) {
    promise = promise.then(chain.shift(), chain.shift()); // 每一次向外弹出拦截器
  }
  // 返回 promise
  return promise;
};
```

可能这段代码看上去用到了```promise```、```interceptors```，有一点迷茫。
那么我们先说```promise```，这是JavaScript为了解决异步回调炼狱问题所提出的一个解决方案，目前已经是前端必备技能，如果有小伙伴不太清楚，自行 Google下？~
然后是```interceptors```，这就是文章开头所说的拦截请求和响应所用到的一个利器————***拦截器***。

##### 拦截器

拦截器的作用是能在我们发起请求之前针对已经流转到此时的 ```config``` 进行再次加工，在我们得到结果之前对已经请求到的结果进行拦截处理。官方给出的时机是 ```then```或者 ```catch``` 之前。

可以看到的是使用拦截器的时候，直接用的```this.interceptors```，那么说明它就是 axios 实例上的一个属性，它的初始化是在 ```Axios.js```，有 `request` 和 `response` 2 个属性。

```js
function Axios(instanceConfig) {
  this.defaults = instanceConfig;
  this.interceptors = {
    request: new InterceptorManager(), // 请求拦截
    response: new InterceptorManager() // 响应拦截
  };
}
```

它的定义是在```InterceptorManager.js```，它们都有一个 `use` 方法，`use` 方法支持 2 个参数，第一个参数类似 Promise 的 `resolve` 函数，第二个参数类似 Promise 的 `reject` 函数。我们可以在 `resolve` 函数和 `reject` 函数中执行同步代码或者是异步代码逻辑。

```js
// 拦截器的初始化 其实就是一组钩子函数
function InterceptorManager() {
  this.handlers = [];
}

// 调用拦截器实例的use时就是往钩子函数中push方法
InterceptorManager.prototype.use = function use(fulfilled, rejected) {
  this.handlers.push({
    fulfilled: fulfilled,
    rejected: rejected
  });
  return this.handlers.length - 1;
};

// 拦截器是可以取消的，根据use的时候返回的ID，把某一个拦截器方法置为null
// 不能用 splice 或者 slice 的原因是 删除之后 id 就会变化，导致之后的顺序或者是操作不可控
InterceptorManager.prototype.eject = function eject(id) {
  if (this.handlers[id]) {
    this.handlers[id] = null;
  }
};

// 这就是在 Axios的request方法中 中循环拦截器的方法 forEach 循环执行钩子函数
InterceptorManager.prototype.forEach = function forEach(fn) {
  utils.forEach(this.handlers, function forEachHandler(h) {
    if (h !== null) {
      fn(h);
    }
  });
};
```

了解了拦截器是什么之后，就要回到```request```中，来看拦截器是如何工作的。之前对于拦截器，我们知道请求拦截器方法是被 ```unshift```到拦截器中，响应拦截器是被```push```到拦截器中的。最终它们会拼接上一个叫```dispatchRequest```的方法被后续的 promise 顺序执行。

拼接的结果是如下的一个链:

```js
[
  请求拦截器2的resolve,
  请求拦截器2的reject,
  请求拦截器1的resolve,
  请求拦截器1的reject,
  dispatchRequest,
  undefined,
  响应拦截器1的resolve,
  响应拦截器1的reject,
  响应拦截器2的resolve,
  响应拦截器2的reject,
]
```

![拦截器](/img/axios/interceptor.png)

因此拦截器的执行顺序是链式依次执行的方式。对于 `request` 拦截器，后添加的拦截器会在请求前的过程中先执行；对于 `response` 拦截器，先添加的拦截器会在响应后先执行。

在构造了这么一个 ```PromiseChain``` 之后，我们可以看到两边是拦截器的处理函数，中间是一个 ```dispatchRequest```，这个是 axios 真真真正发起一个请求的地方，之后会进行详细介绍。

接下来定义一个已经 resolve 了 config 的 promise，循环这个 chain，拿到每个拦截器对象，把它们的 resolved 函数和 rejected 函数添加到 promise.then 的参数中，这样就相当于通过 Promise 的链式调用方式，最终返回一个promise，实现了拦截器一层层的链式调用的效果。

至此前半部分告一段落，接下来会对```dispatchRequest```和数据转换以及取消请求等附属的功能进行分析。
