[Axios](https://github.com/axios/axios)是一款基于Promise的http请求库，小巧玲珑，目前github上stars 82K，支持运行在浏览器端和Nodejs中。能够常年霸占榜位，其源码还是非常值得分析分析的。

## 写在开头
本文是基于源码 V0.21.1版本分析，详细介绍Axios生成实例的过程、构造函数、拦截器、取消等功能，每个小节都会从如何使用到源码逐步解剖。
#### **在分析源码之前，我们先思考几个问题，如果让你写源码，会如何实现？**
1. 为何在浏览器端、Nodejs中都可以使用Axios·？
2. Axios拦截器是如何实现的？
3. 每个开源项目都有自己的工具库，Axios工具库里是否会出现让你拍大腿的实现方式？
4. Axios是怎么实现取消功能的？
5. Axios从请求到结束，都经过了哪些流程，有哪些实现的很巧妙的方式值得学习的？
6. 从使用者的角度看，Axios使用方式有 axios({config})、axios.post()、axios.get() 等等。axios很像一个Object对象，又好像是一个Function，但到底是什么类型的呢？源代码是怎么支持这些请求方式的呢？
```
├── /dist/                     # build输出目录，支持直接script引用 
├── /lib/                      # 项目源码目录
│ ├── /adapters/               # 适配浏览器、nodejs请求
│ │ ├── http.js                # nodejs请求核心
│ │ └── xhr.js                 # 浏览器请求核心
│ ├── /cancel/                 # 实现取消请求的一系列功能
│ ├── /core/                   # 核心功能
│ │ ├── Axios.js               # axios的核心类
│ │ ├── dispatchRequest.js     # 用来调用http请求适配器方法发送请求
│ │ ├── InterceptorManager.js  # 拦截器构造函数
│ │ └── settle.js              # 根据http响应状态，改变Promise的状态
│ ├── /helpers/                # 一些辅助方法
│ ├── axios.js                 # 总入口，生成实例，对外暴露接口
│ ├── defaults.js              # 库默认配置 
│ └── utils.js                 # 公用方法库
├── package.json               # 项目信息
├── index.d.ts                 # 配置TypeScript的声明文件
└── index.js                   # 入口文件
```
## 先举个栗子

在异步请求前先调用指定方法，请求后也调用指定方法进行"过滤"
```js
function myAxios(config){
    this.config = config;
    this.before = [];
    this.after = [];
}
myAxios.prototype.request = function(config){
    return new Promise(resolve => {
        setTimeout(() => {
            resolve(config)
        },3000)
    })
}
```
以上是一个简单的构造函数`myAxios`，原型链上有`request`方法。现在要求在`request`方法执行前，若`before`数组中有方法，则先执行，`after`中若有方法，则在`request`之后继续执行，该如何修改以上代码才能实现？答案其实就是`Promise链式`调用。不熟悉`Promise`的同学建议先看下阮老师的《ES6入门》。接下来我们修改下`request`方法
```js
function myAxios(config){
    this.config = config;
    this.before = [];
    this.after = [];
}
myAxios.prototype.addBeforeFn = function(fn){
    this.before.push(fn)
}
myAxios.prototype.addAfterFn = function(fn){
    this.after.push(fn)
}
function dispatchRequest(config){
    //发送真正请求
    return new Promise(resolve => {
        console.log('发送真正异步请求')
        setTimeout(() => {
            resolve(config)
        },3000)
    })
}
myAxios.prototype.request = function(){
    let promise = Promise.resolve(this.config);
    let chain = [dispatchRequest];
    if(this.before.length) chain.unshift(this.before[0])
    if(this.after.length) chain.push(this.after[0])
    for(let i=0;i<chain.length;i++){
        promise = promise.then(chain[i])
    }
    return promise
}
```
以上为改造后的方法：增加了往实例的属性`before`或者`after`添加属性的方法(注意：添加的方法必须有`return`返回)。原型链上`request`方法中声明一个`chain`属性数组，`dispatchRequest`为真正异步请求。拦截方法分别放入`chain`数组中`dispatchRequest`属性前后，方便`Promise`按照顺序依次`then`
```js
var instance = new myAxios({a:1});
instance.addBeforeFn((config) => {
    console.log('真正请求前')
    return config
})
instance.addAfterFn((config) => {
    console.log('真正请求后')
    return config
})
instance.request().then(res => {
    console.log(res)
})
```
执行后得到的输出是：
```js
// 真正请求前
// 发送真正异步请求
// 真正请求后
// {a:1}
```
这样可以在真正请求前后，对输入或者输出内容做一个拦截处理。当然，在实际应用过程中以上demo拥有很多瑕疵，但在讲解`axios`源码之前举这个例子，就是为了方便后面理解`拦截器`原理

## 工具方法篇
在深入了解`axios`源码之前，有必要先分析下`util.js`中的几个主要工具方法，了解清楚后才更加容易理解源码<br /><br />
1、手动写了一个`bind`方法，给某个方法指定上下文，也就是`this`的指向，生成一个新的方法，实现效果同原生`Function.prototype.bind`
```js
module.exports = function bind(fn, thisArg) {
  return function wrap() {
    var args = new Array(arguments.length);
    for (var i = 0; i < args.length; i++) {
      args[i] = arguments[i];
    }
    return fn.apply(thisArg, args);
  };
};
```
2、`forEach(obj, fn)`使用给定的方法遍历执行给定的数组或对象
```js
function forEach(obj, fn) {
  //省略部分边界代码
  if (isArray(obj)) {
    //数组的处理方式
    for (var i = 0, l = obj.length; i < l; i++) {
      fn.call(null, obj[i], i, obj);
    }
  } else {
    //对象的处理方式
    for (var key in obj) {
      if (Object.prototype.hasOwnProperty.call(obj, key)) {
        fn.call(null, obj[key], key, obj);
      }
    }
  }
}
```
3、`merge(/* obj1, obj2, obj3, ... */)`深度合并多个对象为一个新对象
```js
function merge(/* obj1, obj2, obj3, ... */) {
  var result = {};
  function assignValue(val, key) {
    if (isPlainObject(result[key]) && isPlainObject(val)) {
      result[key] = merge(result[key], val);
    } else if (isPlainObject(val)) {
      result[key] = merge({}, val);
    } else if (isArray(val)) {
      result[key] = val.slice();
    } else {
      result[key] = val;
    }
  }

  for (var i = 0, l = arguments.length; i < l; i++) {
    forEach(arguments[i], assignValue);
  }
  return result;
}
```
思考：如果工作中我们遇到合并多个对象为一个新对象的情况，会如何处理？<br /><br />
3、`extend(a, b, thisArg)`将`b`对象中的方法或属性扩展到`a`中，并指定上下文
```js
function extend(a, b, thisArg) {
  forEach(b, function assignValue(val, key) {
    if (thisArg && typeof val === 'function') {
      a[key] = bind(val, thisArg);
    } else {
      a[key] = val;
    }
  });
  return a;
}
```
> 评论：以上三个方法，虽然看起来很简单，但非常有意思，值得我们手动实现感受一下，相信会理解的更加深刻。

## 入口文件`lib/axios.js`
引入工具函数`util`、绑定函数`bind`、默认配置`defaults`、构造函数`Axios`、合并配置函数`mergeConfig`等

```js
var utils = require('./utils');
var bind = require('./helpers/bind');
var Axios = require('./core/Axios');
var mergeConfig = require('./core/mergeConfig');
var defaults = require('./defaults');
```
生成实例对象方法`axios`
```js
function createInstance(defaultConfig) {
  //生成一个实例对象context，包含属性defaults(默认配置)、interceptors(拦截器)
  var context = new Axios(defaultConfig);
  //bind方法在工具篇已有介绍
  //其实所有的请求，都是走的Axios.prototype.request方法
  //绑定返回一个新Function，指定this的指向为context，这也就是axios(config)可以使用的原因
  var instance = bind(Axios.prototype.request, context);
  //复制 Axios.prototype原型的方法到实例上，并绑定this的指向到context
  //这就是为什么可以使用axios.post、axios.get等别名的原因
  utils.extend(instance, Axios.prototype, context);
  //复制context的属性到实例instance上
  //这就是为啥实际使用的时候axios.defaults、axios.interceptors
  utils.extend(instance, context);
  //返回实例对象(其实是一个方法)
  return instance;
}
var axios = createInstance(defaults);
```
暴露取消API等其他方法
```js
//暴露核心类库
axios.Axios = Axios;
//暴露 工厂模式 创建实例，满足个性化用户需求
axios.create = function create(instanceConfig) {
  return createInstance(mergeConfig(axios.defaults, instanceConfig));
};
//* 以上两者一般使用较少 *//

// 导出 Cancel 和 CancelToken
axios.Cancel = require('./cancel/Cancel');
axios.CancelToken = require('./cancel/CancelToken');
axios.isCancel = require('./cancel/isCancel');
//其实就是Promise.all方法
axios.all = function all(promises) {
  return Promise.all(promises);
};
//导出 spread
axios.spread = require('./helpers/spread');
//导出错误捕获 isAxiosError
axios.isAxiosError = require('./helpers/isAxiosError');
```
## `axios`到底是何种类型以及暴露的方法、属性
其实从`axios(config)`可以调用就可以发现，`axios`是一个方法
```js
Object.prototype.toString.call(axios)
// [object Function]
```
测试一下`axios`本身暴露的方法以及属性
```js
console.log(Object.keys(axios))
//['request', 'getUri', 'delete', 'get', 'head', 'options', 'post', 'put', 'patch', 'defaults', 'interceptors', 'Axios', 'create', 'Cancel', 'CancelToken', 'isCancel', 'all', 'spread', 'default']
```
## 核心类库`lib/Axios.js`
1、引入工具方法、拦截器、实际发送请求方法等
```js
var utils = require('./../utils');
var buildURL = require('../helpers/buildURL');
var InterceptorManager = require('./InterceptorManager');
var dispatchRequest = require('./dispatchRequest');
var mergeConfig = require('./mergeConfig');
```
2、核心构造函数`Axios`(拦截器方法`InterceptorManager`我们稍后分析)
```js
function Axios(instanceConfig) {
  //defaults参数保存默认配置
  this.defaults = instanceConfig;
  //创建请求拦截器、相应拦截器
  this.interceptors = {
    request: new InterceptorManager(),
    response: new InterceptorManager()
  };
}
```
3、核心请求方法。不论使用`axios`、`axios.post`、还是`axios.get`等方式请求，最终实际请求的方法都是原型链上的`request`方法。
> 总结：`request`方法中真正的请求，其实就是`dispatchRequest`真正请求接口，其他的一些操作，其实是为了实现请求前后的拦截，应用原理其实就是`Promise.then()`链式调用。若你已看懂前面例子，这里理解起来将会更加容易。
```js
Axios.prototype.request = function request(config) {
  //忽略一些边界、异常情况处理代码
    
  //创建一个数组，成对保存promise回调方法(成对出现，一个是成功，一个是失败，这也是初始化数组的时候，第二个参数要设置为undefined的原因)
  var chain = [dispatchRequest, undefined];
  var promise = Promise.resolve(config);
  //遍历执行，将请求拦截方法添加到数组chain前部
  this.interceptors.request.forEach(function unshiftRequestInterceptors(interceptor) {
    chain.unshift(interceptor.fulfilled, interceptor.rejected);
  });
  //遍历执行，将响应拦截方法添加到数组chain尾部
  this.interceptors.response.forEach(function pushResponseInterceptors(interceptor) {
    chain.push(interceptor.fulfilled, interceptor.rejected);
  });
  //遍历数组，组成Promise链式格式
  while (chain.length) {
    promise = promise.then(chain.shift(), chain.shift());
  }
  return promise;
};
```
`request`方法最终返回的是一个`Promise`链式调用，最终返回的格式大概如下
```js
//返回的格式大概是 
  Promise.resolve(config).then(config => {
    //请求前拦截
  }).then(config => {
    //真正请求
  }).then(config => {
    //请求后相应拦截
  })
```
获取URL请求的函数(个人没太理解这个函数用处多大，懂的小伙伴可指教指教)
```js
Axios.prototype.getUri = function getUri(config) {
  config = mergeConfig(this.defaults, config);
  return buildURL(config.url, config.params, config.paramsSerializer).replace(/^\?/, '');
};
```
在`Axios`原型链上遍历添加别名方法`delete`、`get`、`head`、`options`、`post`、`put`、`patch`。在生成实例的方法中，`Axios`原型链的方法都复制拷贝作为`axios`属性方法，所以可以使用`axios.post`、`axios.get`等方式请求
```js
utils.forEach(['delete', 'get', 'head', 'options'], function forEachMethodNoData(method) {
  Axios.prototype[method] = function(url, config) {
    return this.request(mergeConfig(config || {}, {
      method: method,
      url: url,
      data: (config || {}).data
    }));
  };
});

utils.forEach(['post', 'put', 'patch'], function forEachMethodWithData(method) {
  Axios.prototype[method] = function(url, data, config) {
    return this.request(mergeConfig(config || {}, {
      method: method,
      url: url,
      data: data
    }));
  };
});
```
## 拦截器构造函数`lib/cor/InterceptorManager.js`
1、如何使用？<br />
拦截器分为请求前拦截器，请求后拦截器
```js
//请求前拦截器使用方法
axios.interceptors.request.use(config => {
  return config
},err => {
  Promise.reject(err)
})

//请求后拦截器
axios.interceptors.response.use(response => {
  return response
},err => {
  
})
```
2、源码分析
```js
// 增加拦截器操作数组，保存拦截函数
function InterceptorManager() {
  this.handlers = [];
}
//添加成功或者失败函数(成对出现)
InterceptorManager.prototype.use = function use(fulfilled, rejected) {
  this.handlers.push({
    fulfilled: fulfilled,
    rejected: rejected
  });
  return this.handlers.length - 1;
};
//移除拦截器方法
InterceptorManager.prototype.eject = function eject(id) {
  if (this.handlers[id]) {
    this.handlers[id] = null;
  }
};
//遍历执行所有拦截器，传递一个函数调用。若是null，则不遍历
InterceptorManager.prototype.forEach = function forEach(fn) {
  utils.forEach(this.handlers, function forEachHandler(h) {
    if (h !== null) {
      fn(h);
    }
  });
};
```
一般来说，添加函数`use`经常用，后两者用的较少，了解原理即可。
## 默认配置`lib/defaults.js`
1、适配器。在设计模式中被称为适配器模式，以`电压转换器`为例，经常全球出差的人知道的，每个国家的电压不同，而我们的电器需要的电压是220V，有了转换器后，只管使用即可，至于输入多少V，则不需要关心
```js
//若无ContentType，则设置默认值，这个很好理解
function setContentTypeIfUnset(headers, value) {
    if (!utils.isUndefined(headers) && utils.isUndefined(headers['Content-Type'])) {
        headers['Content-Type'] = value;
    }
}

//请求适配器
//这里解释了axios同时支持浏览器以及nodejs环境的原因
//若是浏览器环境，则使用xhr.js 否则若是nodejs环境，则使用http.js
function getDefaultAdapter() {
    var adapter;
    if (typeof XMLHttpRequest !== 'undefined') {
        // For browsers use XHR adapter
        adapter = require('./adapters/xhr');
    } else if (typeof process !== 'undefined' && Object.prototype.toString.call(process) === '[object process]') {
        // For node use HTTP adapter
        adapter = require('./adapters/http');
    }
    return adapter;
}
```
2、其他默认配置
```js
var defaults = {
    //适配器
    adapter: getDefaultAdapter(),
    //请求转换器
    transformRequest: [function transformRequest(data, headers) {
        
    }],
    //请求后数据转换器
    transformResponse: [function transformResponse(data) {
        
    }],
    //默认超时时间为0，意味着无超时时间
    timeout: 0,

    xsrfCookieName: 'XSRF-TOKEN',
    xsrfHeaderName: 'X-XSRF-TOKEN',

    maxContentLength: -1,
    maxBodyLength: -1,

    validateStatus: function validateStatus(status) {
        return status >= 200 && status < 300;
    }
};
```
## 派发请求方法`dispatchRequest`
```js
module.exports = function dispatchRequest(config) {
  //如果config中存在cancelToken,则throw 错误，从而使得Promise走向错误
  throwIfCancellationRequested(config);

  // 确保存在headers头
  config.headers = config.headers || {};

  // 转换请求参数
  config.data = transformData(
    config.data,
    config.headers,
    config.transformRequest
  );

  // 拍平头
  config.headers = utils.merge(
    config.headers.common || {},
    config.headers[config.method] || {},
    config.headers
  );
  //删除一些无用的头信息
  utils.forEach(
    ['delete', 'get', 'head', 'post', 'put', 'patch', 'common'],
    function cleanHeaderConfig(method) {
      delete config.headers[method];
    }
  );
  //优先使用配置中的适配器，若没有则使用默认适配器
  //这里也提醒了我们，若不高兴使用默认的请求方式，也可以自己配置
  var adapter = config.adapter || defaults.adapter;

  return adapter(config).then(function onAdapterResolution(response) {
    //让请求返回Promise.reject()，从而达到取消请求的目的
    throwIfCancellationRequested(config);

    // 转换请求结果
    response.data = transformData(
      response.data,
      response.headers,
      config.transformResponse
    );

    return response;
  }, function onAdapterRejection(reason) {
    if (!isCancel(reason)) {
      //取消相关
      throwIfCancellationRequested(config);
      //转换响应数据
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
总结`dispatchRequest`主要做了些啥：除了一些边界处理之外，就是取消相关的API操作，真正请求前后的数据转换
## 浏览器原生`XMLHttpRequest`方法
位置:`lib/adapters/xhr.js`。在浏览器中真正的请求，都是会依赖并调用原生`XMLHttpRequest`方法去执行的，`axios`其实说到底也只是对`XMLHttpRequest`的封装，围绕请求做了一些边界处理，以及更好的开发体验而已，从而让开发者无需关心底层是如何请求的，只需要将请求参数、请求地址配置好坐等结果即可。
```js
module.exports = function xhrAdapter(config) {
  return new Promise(function dispatchXhrRequest(resolve, reject) {
    //忽略一些边界处理方式
    var request = new XMLHttpRequest();
    //获取请求全连接
    var fullPath = buildFullPath(config.baseURL, config.url);
    //配置请求
    request.open(config.method.toUpperCase(), buildURL(fullPath, config.params, config.paramsSerializer), true);
    //设置超时时间
    request.timeout = config.timeout;
    //监听请求状态
    request.onreadystatechange = function handleLoad() {
      
    };
    // 处理浏览器请求取消（与手动取消相反
    request.onabort = function handleAbort() {
      
    };
    // 捕获请求过程中产生的错误
    request.onerror = function handleError() {
      
    };
    // 超时请求
    request.ontimeout = function handleTimeout() {
      
    };
    // 只有在标准浏览器中，才会添加可能的xsrf头。何为标准浏览器内，可移步util.js中查看isStandardBrowserEnv方法，相对简单，此处不再赘述
    if (utils.isStandardBrowserEnv()) {
      //添加 xsrf 头
      var xsrfValue = (config.withCredentials || isURLSameOrigin(fullPath)) && config.xsrfCookieName ?
        cookies.read(config.xsrfCookieName) :
        undefined;
      if (xsrfValue) {
        requestHeaders[config.xsrfHeaderName] = xsrfValue;
      }
    }
    // 是否允许 跨站携带cookie
    if (!utils.isUndefined(config.withCredentials)) {
      request.withCredentials = !!config.withCredentials;
    }
    // 若配置中将config.responseType设置为true，则将responseType添加到请求
    if (config.responseType) {
      try {
        request.responseType = config.responseType;
      } catch (e) {
        
      }
    }
    // 若提前配置了onDownloadProgress，则监听处理进度
    if (typeof config.onDownloadProgress === 'function') {
      request.addEventListener('progress', config.onDownloadProgress);
    }

    // Not all browsers support upload events
    if (typeof config.onUploadProgress === 'function' && request.upload) {
      request.upload.addEventListener('progress', config.onUploadProgress);
    }
    //处理取消请求相关
    if (config.cancelToken) {
      config.cancelToken.promise.then(function onCanceled(cancel) {
        if (!request) {
          return;
        }
        request.abort();
        reject(cancel);
        request = null;
      });
    }

    if (!requestData) {
      requestData = null;
    }
    // 发送请求
    request.send(requestData);
  });
};
```
## 取消API如何使用以及源码分析
1、取消请求的原理，其实就是让`Promise throw`一个错误，从而阻断链式调用达到目的。取消示例：
```js
const source = axios.CancelToken.source();
axios.post('/yourServerAddress', {
  cancelToken: source.token
}).catch(err => {
  if (axios.isCancel(err)) {
    console.log('请求取消，原因是：', err.message);
  }else{
   
  }
});
// 取消函数
source.cancel('因某种原因，请求被取消');

//* 或者这样取消(个人更推荐后者写法，这样显得直观，且更像一个整体，方便阅读) *//
axios.post('/yourServerAddress',{
  cancelToken:new axios.CancelToken(cancel => {
    if(/* 某种原因 */){
      //取消请求
    }
  })
})

```
2、`取消请求`源码分析，位置:`lib/cancel/CancelToken.js`，关键代码：
```js
function CancelToken(executor) {
  // ....
  //忽略一些边界处理方式
  
  //取消关键点：得到实例属性promise，此时请求状态为pendding中，而xhr.js中有这么一行关键代码，当发现存在cancelToken，则abort()原生XMLHttpRequest，且当前Promise.reject，中断链式调用
  
  //xhr.js中取消相关代码(乱入)
  if (config.cancelToken) {
    config.cancelToken.promise.then(function onCanceled(cancel) {
      request.abort();
      reject(cancel);
      request = null;
    });
  }
    
  var resolvePromise;
  this.promise = new Promise(function promiseExecutor(resolve) {
    resolvePromise = resolve;
  });

  var token = this;
  executor(function cancel(message) {
    if (token.reason) {
      return;
    }
    token.reason = new Cancel(message);
    resolvePromise(token.reason);
  });
}

//...忽略部分不太重要的代码

CancelToken.source = function source() {
  var cancel;
  var token = new CancelToken(function executor(c) {
    cancel = c;
  });
  return {
    token: token,
    cancel: cancel
  };
};
```
## 总结
至此，`axios`源码基本分析完毕。个人认为其中有这么几点非常值得学习，其想法值得借鉴到我们的项目中<br />
1、工具方法之`forEach`、`merge`、`extend`，短短几行代码，写的非常巧妙；<br />
2、生成`axios`实例的时候，拷贝合并暴露原型链方法；<br />
3、`Axios.prototype.request`中使用`Promise`链式调用实现的请求前后拦截；<br />
4、原生方法`XMLHttpRequest`对请求的处理等一系列操作；<br />
5、取消API的实现
