---
title: axios 源码简析(二)
index_img: /img/axios/axios.png
tags: JavaScript三方库
date: 2019-08-06 20:20:30
---

Promise based HTTP client for the browser and node.js —— 基于 Promise 的 HTTP 库，可以用在浏览器和 NodeJs 中
<!-- more -->

在之前的[上一篇](/2019/08/04/axios源码简析(一)/)已经对 axios 的调用方式、拦截器等进行了分析说明。这次主要对 ```dispatchRequest```发起请求、取消请求、数据转换和XSRF防御进行简单分析。

#### dispatchRequest

在拦截器的调用链中，一开始就放置的一个调用函数就是```dispatchRequest```，它的定义是在 ```core/dispatchRequest.js```，以下是源码:

```js
var transformData = require('./transformData');

// 取消请求的异常处理
function throwIfCancellationRequested(config) {
  if (config.cancelToken) {
    config.cancelToken.throwIfRequested();
  }
}

module.exports = function dispatchRequest(config) {
  throwIfCancellationRequested(config);

  // Support baseURL config 支持 baseURL 的拼接
  if (config.baseURL && !isAbsoluteURL(config.url)) {
    config.url = combineURLs(config.baseURL, config.url);
  }

  // Ensure headers exist 处理headers
  config.headers = config.headers || {};

  // Transform request data 请求数据转换
  config.data = transformData(
    config.data,
    config.headers,
    config.transformRequest
  );

  // headers 的修改
  // Flatten headers
  config.headers = utils.merge(
    config.headers.common || {},
    config.headers[config.method] || {},
    config.headers || {}
  );

  // adapter 是适配器意思，在axios里，adapter有两种： 1、xhr 2、nodejs的http(https)
  // 浏览器环境的 adapter 就是 xhr，稍后会说明 adapter 逻辑
  var adapter = config.adapter || defaults.adapter;

  // 发起请求 promise
  return adapter(config).then(function onAdapterResolution(response) {
    // 请求成功的回调
    throwIfCancellationRequested(config);

    // Transform response data 响应数据转换
    response.data = transformData(
      response.data,
      response.headers,
      config.transformResponse
    );

    return response;
  }, function onAdapterRejection(reason) {
    // 请求失败的回调
    if (!isCancel(reason)) {
      throwIfCancellationRequested(config);

      // Transform response data 响应数据转换
      if (reason && reason.response) {
        reason.response.data = transformData(
          reason.response.data,
          reason.response.headers,
          config.transformResponse
        );
      }
    }
    return Promise.reject(reason);
  });
};
```

可以看到在发起请求的时候，又去操作了```config```，且多次的执行```transformData```函数对请求数据以及响应数据做处理，这就是***数据转换***，最终```return```了一个```adapter```。

emmmm...先中间插一下```adapter```吧，稍后再说```transformData```数据转换。

##### adapter

```adapter```是挂载在```config```上的，一般我们不会去指定这个配置，使用默认的即可，那么它是默认定义在 ```defaults.js```中:

```js
function getDefaultAdapter() {
  var adapter;
  // Only Node.JS has a process variable that is of [[Class]] process nodeJS环境判断
  if (typeof process !== 'undefined'
    && Object.prototype.toString.call(process) === '[object process]'
  ) {
    // For node use HTTP adapter
    adapter = require('./adapters/http');
  } else if (typeof XMLHttpRequest !== 'undefined') { // 浏览器环境判断
    // For browsers use XHR adapter
    adapter = require('./adapters/xhr');
  }
  return adapter;
}

var defaults = {
  adapter: getDefaultAdapter(),
  /*
    something ...
  */
}
module.exports = defaults;
```

```getDefaultAdapter```方法的内部判断了当前的 axios 执行环境，根据环境的不同引入了不同的```adapter```，本次只分析浏览器环境的实现，所以```/adapters/xhr.js```是我们所关心的，来简单看一下内部实现:

```js
module.exports = function xhrAdapter(config) {
  return new Promise(function dispatchXhrRequest(resolve, reject) {
    /* something ... 获取参数 */

    // 熟悉的 XMLHttpRequest
    var request = new XMLHttpRequest();

    /* something ... auth 等 */

    request.open(config.method.toUpperCase(), buildURL(config.url, config.params, config.paramsSerializer), true);

    request.onreadystatechange = function handleLoad() {
      if (!request || request.readyState !== 4) {
        return;
      }
      if (request.status === 0 && !(request.responseURL && request.responseURL.indexOf('file:') === 0)) {
        return;
      }

      /* something ... */

      // 更改封装的promise状态
      settle(resolve, reject, response);
    };

    /*
     中间部分是一些事件的监听，比如: 取消、超时、错误、请求进度、xsrf、取消请求等
    */
    request.send(requestData);
  });
};
```

喏！~应该已经看到了作为前端开发者熟悉的```XMLHttpRequest```。

##### transformData

回到```transformData```数据转换，其实 axios 的**JSON转换**功能就是在这里实现的，来看一下```transformData```的定义吧:

```js
// transformData.js
module.exports = function transformData(data, headers, fns) {
  // 循环调用传入的 fns 去处理传入的 data, headers
  utils.forEach(fns, function transform(fn) {
    data = fn(data, headers);
  });
  // 返回 data
  return data;
};
```

其实就是循环调用传入的```config.transformResponse```或者```config.transformRequest```去对```data```和```headers```做处理，最终返回经过处理的```data```，不过请求前处理的是请求参数data，请求后处理的是请求结果data。

那么就来揭开```config.transformResponse```或者```config.transformRequest```的面纱吧:

```js
// defaults.js
var defaults = {
  transformRequest: [function transformRequest(data, headers) {
    // 格式化 headers 的 Accept 和 Content-Type
    normalizeHeaderName(headers, 'Accept');
    normalizeHeaderName(headers, 'Content-Type');
    // 如果是 FormData ArrayBuffer Buffer Stream File Blob 就不处理直接返回
    if (utils.isFormData(data) ||
      utils.isArrayBuffer(data) ||
      utils.isBuffer(data) ||
      utils.isStream(data) ||
      utils.isFile(data) ||
      utils.isBlob(data)
    ) {
      return data;
    }
    // 如果是 ArrayBuffer 视图模型则取出 ArrayBuffer 实体内容
    if (utils.isArrayBufferView(data)) {
      return data.buffer;
    }
    // 如果是 对象 转化成字符串
    if (utils.isObject(data)) {
      setContentTypeIfUnset(headers, 'application/json;charset=utf-8');
      return JSON.stringify(data);
    }
    return data;
  }],

  transformResponse: [function transformResponse(data) {
    // 自动转换成JSON就在这里啦~~~判断是否是字符串，尝试转换JSON
    if (typeof data === 'string') {
      try {
        data = JSON.parse(data);
      } catch (e) { /* Ignore 忽略处理，否则会报错，得不到结果 */ }
    }
    return data;
  }],
  /* something... */
}
```

可以看到的是数据转换是 axios 的一个默认处理行为，在之前我们得知用户在使用 axios 的时候，自定义配置的优先级是高于默认配置的优先级的，所以如果在实战中使用的时候需要注意 __如果不确定是否要舍弃默认的数据转换行为，在覆盖的时候默认行为添加上自定义转换配置__:

例如:

```js
axios({
  transformRequest: [
        function(data) {
            /* do something */
            return data;
        },
        ...(axios.defaults.transformRequest)
  ],
  transformResponse: [
      ...(axios.defaults.transformResponse),
      function(data) {
        /* do something */
        return data;
      }
  ],
  url: 'xxxx',
  data: {}
}).then((res) => {
  console.log(res.data)
})
```

#### 取消请求

axios 在实现取消请求的时候是很巧妙的运用了 Promise 的特性的，深感 JavaScript 的强大中...接下来就来一探究竟吧

先来看下使用 axios 中是如何取消请求的发送的:

```js
// 方式1
const CancelToken = axios.CancelToken;
const source = CancelToken.source();

axios.get('xxxx', {
  cancelToken: source.token
})
// 取消请求 (请求原因是可选的)
source.cancel('主动取消请求');

// 方式二
const CancelToken = axios.CancelToken;
let cancel;

axios.get('xxxx', {
  cancelToken: new CancelToken(function executor(c) {
    cancel = c;
  })
});
cancel('主动取消请求');
```

针对以上两种方式的取消都是在```CancelToken.js```中实现的，具体代码如下:

```js
function CancelToken(executor) {
  if (typeof executor !== 'function') {
    throw new TypeError('executor must be a function.');
  }
  // 在 CancelToken 上定义一个 pending 状态的 promise ，将 resolve 回调赋值给外部变量 resolvePromise
  var resolvePromise;
  this.promise = new Promise(function promiseExecutor(resolve) {
    resolvePromise = resolve;
  });

  var token = this;
  // 立即执行 传入的 executor函数，将真实的 cancel 方法通过参数传递出去。
  // 一旦调用就执行 resolvePromise 即前面的 promise 的 resolve，就更改promise的状态为 resolve。
  // 那么xhr中定义的 CancelToken.promise.then方法就会执行, 从而xhr内部会取消请求
  executor(function cancel(message) {
    // 判断请求是否已经取消过，避免多次执行
    if (token.reason) {
      return;
    }
    token.reason = new Cancel(message);
    resolvePromise(token.reason);
  });
}

CancelToken.source = function source() {
  // source 方法就是返回了一个 CancelToken 实例，与直接使用 new CancelToken 是一样的操作
  var cancel;
  var token = new CancelToken(function executor(c) {
    cancel = c;
  });
  // 返回创建的 CancelToken 实例以及取消方法
  return {
    token: token,
    cancel: cancel
  };
};
```

实际上取消请求的操作是在 ```xhr.js``` 中也有响应的配合的

```js
if (config.cancelToken) {
    config.cancelToken.promise.then(function onCanceled(cancel) {
        if (!request) {
            return;
        }
        // 取消请求
        request.abort();
        reject(cancel);
    });
}
```

巧妙的地方在 ```CancelToken```中 ```executor``` 函数，通过```resolve```函数的传递与执行，控制一个```promise```的状态。

### 结束

到此 axios 的大部分流程实现已经简单分析完成。如有表述的不合理或者是错误的地方，欢迎小伙伴指正。
