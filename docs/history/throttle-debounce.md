# 告别死概念，用大白话理解“防抖与节流”
网上关于防抖与节流的文章，大概率都是互相抄一下，基本上千篇一律。今天我从小白的角度聊聊这两者的特点以及差异。
## 一、应用场景
- input输入框，用户频繁操作输入框内容匹配关键词或者请求接口。
- 窗口`resize`或者页面`scroll`滚动调整后需要回调做的事情，例如图片懒加载、页面组件大小变动等。
- `socket`推送数据，例如股票买卖盘口数据，并不是后台推送啥就完全展示啥，适当的漏掉一部分也是合理的。例如100ms内后台推送了几百条数据，实时操作DOM不仅会带来性能上的浪费，造成页面卡顿、假死，而且对用户感官上来说200ms渲染一次跟实时渲染，区别不是很大。
- `mousedown`、`mousemove`、`keyup`、`keydown`等频繁的鼠标操作。
- 其他需要限制频率，不需要实时操作的场景。
防抖与节流的共同特点是用于高频次触发的场景，将`高频`降为`低频`达到性能优化的目的。区别是前者用一定手段将`最后一次`操作落地执行，后者是间隔`300ms`时间内最多执行一次。
## 二、什么是防抖
以`input`输入框输入内容并请求http接口为例。用户输入`123456789`，若不做其他处理，实时监听`input`输入内容，变动一次就请求一次接口，这个时候会产生9次请求。若同时有10W个用户在线同时操作，对后台同学的压力可想而知，而且有很多请求都是浪费的，同时还有可能产生第一次请求接口返回的数据要晚于第五次请求返回的情况。这个时候就需要做`防抖`的优化：实时监听输入数据变化，只要有300ms时间没有变化，则请求接口。也就是300ms没变动，则认为用户要搜索的关键词就是当前内容。这个时候你大概就能了解什么是防抖了：在一定时间内，没有再进行频繁操作，目的是尽量确保频繁操作的最后一次才实际执行回调，其他的视为`无效`
```js
//一个简单版的防抖
var timer = null;
function debounce(){
  if(timer) clearTimeout(timer);
  timer = setTimeout(function(){
    console.log('窗口大小变化啦')
  },300)
}
window.addEventListener('resize',debounce);
```
以上代码的意思是当你在放大或者缩小浏览器窗口大小的时候，只有停止操作后300ms之后，才会真正的执行相关操作，这样在很大程度上避免不必要的浪费。<br />
上面的写法可以很形象的帮助理解`防抖`，但需要实际封装一下，毕竟这么`low`的代码体现不出高端的水平，更不适合页面多个地方调用，利用`闭包`适当封装一下，方便多处地方随意调用
```js
function debounce(fn,delay){
  var timer = null;
  return function(){
    if(timer) clearTimeout(timer);
    timer = setTimeout(function(){
      fn.apply(this)
    },delay)
  }
}
window.addEventListener('resize',debounce(function(){
  console.log('窗口大小变化啦')
},300));
```
上面例子中，实际`this`值总是指向`window`(浏览器端)，若要根据实际场景，并指向执行环境且能够传参的话，再做适当的修改
```js
function debounce(fn,delay){
  var timer = null;
  return function(){
    if(timer) clearTimeout(timer);
    var self = this,args = arguments;
    timer = setTimeout(function(){
      fn.apply(self,args)
    },delay)
  }
}
window.addEventListener('resize',debounce(function(e){
  console.log(this)
  console.log(e)
},300));
```
这里有的同学可能会有一个疑问：为何是`fn.call(self,args)`，而不是直接`fn(args)`？其实关键在于第一个参数，确保上下文为当前的`this`。
### 思考一：如何控制`防抖`函数，在一开始的时候也执行一次？
什么意思呢？就是再增加一个开关，我们知道`防抖`是执行的频繁操作的最后一次，但是我想频繁操作的第一次和最后一次都执行。这个想法没毛病的，比如频繁操作了5秒才能看到结果，这5秒内等于是空白的，用户也许看到的是一个空白的东西，如果刚开始就执行一次，呈现内容出来，好过一直空白的好。<br />
实现思路：增加一个开关，若需要立即执行且开关是允许的时候则立即执行，不再延时，同时立马关闭开关。
```js
function debounce(fn,delay,immediate){
  var timer = null;
  var isImmediate = false;
  return function(){
    if(!isImmediate && immediate){
      isImmediate = immediate;
      fn.apply(this,arguments)
    }else{
      if(timer) clearTimeout(timer);
      var self = this,args = arguments;
      timer = setTimeout(function(){
        fn.apply(self,args)
      },delay)
    }
  }
}
window.addEventListener('resize',debounce(function(e){
  console.log(this)
  console.log(e)
},300,true));
```
### 思考二：如何取消延时执行？
我们知道`防抖`是在最后一次事件操作后延迟N秒执行的，假如我们设置的延时时间为5秒，那么频繁操作结束后我们要等待5秒才会执行，因一些特殊情况，比如有另外一个按钮触发一下则取消延时以及后续操作，这个如何实现呢？在上面的基础上我们继续
```js
function debounce(fn,delay,immediate){
  var timer = null;
  var isImmediate = false;
  var de = function(){
    if(!isImmediate && immediate){
      isImmediate = immediate;
      fn.apply(this,arguments)
    }else{
      if(timer) clearTimeout(timer);
      var self = this,args = arguments;
      timer = setTimeout(function(){
        fn.apply(self,args)
      },delay)
    }
  }
  de.cancel = function(){
    clearTimeout(timer);
    timer = null;
    console.log('已经取消啦')
  }
  return de
}
var resizeFn = debounce(function(e){
  console.log(e)
},5000,true)
window.addEventListener('resize',resizeFn);
//取消操作
document.querySelector('.calcenButton').addEventListener('click',function(){
  resizeFn.cancel();
})
```
## 三、什么是节流
以股票中`socket`推送买卖盘口为例，假设后台1s内推送了1w次数据，我们若把这1w次都渲染到页面，想必页面会假死掉，同时也会大量耗费浏览器性能。若修改为每隔300ms获取一次数据源并渲染到页面，1s内最多也就渲染三次，性能大大降低，这就是`节流`。用大白话描述就是：事件你尽管执行，我只间隔一段时间执行一次。说到这里，感觉已经可以写出`节流`函数了：
```js
function throttle(fn,wait){
  var pre = 0;
  return function(){
    var self = this,args = arguments;
    var now = +new Date();
    if(now - pre > wait){
      pre = now;
      fn.apply(self,args)
    }
  }
}
window.addEventListener('resize',throttle(function(){
  console.log('窗口大小变化啦')
},1000));
```
以上是使用时间戳，配合事件操作驱动的`节流`，下面我们换一个定时器实现的方式。实现的思路是：设定一个定时器，每次执行的时候若定时器存在则忽略，若不存在则再设置一个定时器，定时器若执行了，则将定时器清空后重新定时。
```js
function throttle(fn,wait){
  var timer = null;
  return function(){
    var self = this,args = arguments;
    if(!timer){
      timer = setTimeout(function(){
        timer = null;
        fn.apply(self,args)
      },wait)
    }
  }
}
window.addEventListener('resize',throttle(function(){
  console.log('窗口大小变化啦')
},1000));
```
以上是常见的两种实现`节流`的方式，区别在于`时间戳节流`会立即触发执行，后续超过`wait`倍数部分若停止事件，则会被`忽略`。而`定时器节流`总是会延迟`wait`时间执行，若停止事件操作，则还有可能会延迟一定时间再执行一次。