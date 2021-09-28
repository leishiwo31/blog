### debug.js
> 正所谓磨刀不误砍柴工，在真正阅读源码之前先了解清楚一些变量是如何判断、纯函数的作用是十分有必要的。知道锤子、斧头、扳手、剪刀是干嘛用的，这样才不至于在初次见面的时候显得措手不及。

该文件主要暴露了四个方法`warn`、`tip`、`generateComponentTrace`、`formatComponentName`，主要代码如下
```js
//noop是从shared/util中引入的空方法
export function noop (a?: any, b?: any, c?: any) {}

//debug.js
export let warn = noop
export let tip = noop
export let generateComponentTrace = (noop: any) // work around flow check
export let formatComponentName = (noop: any)
if (process.env.NODE_ENV !== 'production') {
	//暂时忽略此部分
}
```
主要在调试中使用，生产环境下他们就仅仅是个空`Function`，不做任何处理，只有在开发环境下这四个变量才会被重新定义。<br />
> 注：源码中以后会经常遇到`process.env.NODE_ENV !== 'production'`这样的写法，这是从性能角度考虑，一些检测、警告等非必要的提示只有在非生产环境才会做处理，以此达到最优性能。
```js
//检测当前环境是否支持 console
const hasConsole = typeof console !== 'undefined'
//正则表达式
const classifyRE = /(?:^|[-_])(\w)/g
//转换字符串格式
const classify = str => str
    .replace(classifyRE, c => c.toUpperCase())
    .replace(/[-_]/g, '')
```
前面两个相对好理解，`classify`的目的是使用正则`classifyRE`将字符串中横岗写法转换成驼峰式。例如：
```js
classify('fishing-wang');
//输出 FisingWang
```
```js
//重写打印或警告内容到客户端
warn = (msg, vm) => {
  const trace = vm ? generateComponentTrace(vm) : ''
  if (config.warnHandler) {
    config.warnHandler.call(null, msg, vm, trace)
  } else if (hasConsole && (!config.silent)) {
    console.error(`[Vue warn]: ${msg}${trace}`)
  }
}

tip = (msg, vm) => {
  if (hasConsole && (!config.silent)) {
    console.warn(`[Vue tip]: ${msg}` + (
      vm ? generateComponentTrace(vm) : ''
    ))
  }
}
```
