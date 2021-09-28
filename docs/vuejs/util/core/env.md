### env.js
> 正所谓磨刀不误砍柴工，在真正阅读源码之前先了解清楚一些变量是如何判断、纯函数的作用是十分有必要的。知道锤子、斧头、扳手、剪刀是干嘛用的，这样才不至于在初次见面的时候显得措手不及。

检测当前宿主环境对变量的支持情况。
```js
//检测当前浏览器是否支持 '__proto__'
export const hasProto = '__proto__' in {}
//检测是否浏览器环境
export const inBrowser = typeof window !== 'undefined'
//检测当前是否是weex环境
export const inWeex = typeof WXEnvironment !== 'undefined' && !!WXEnvironment.platform
//是否weex平台
export const weexPlatform = inWeex && WXEnvironment.platform.toLowerCase()
//先确保当前环境是浏览器环境，然后将浏览器的use Agent信息保存在 UA 变量中 (非浏览器环境则直接返回false)
export const UA = inBrowser && window.navigator.userAgent.toLowerCase()
//是否IE浏览器
export const isIE = UA && /msie|trident/.test(UA)
//是否IE9浏览器
export const isIE9 = UA && UA.indexOf('msie 9.0') > 0
//是否Edge浏览器
export const isEdge = UA && UA.indexOf('edge/') > 0
//是否安卓环境
export const isAndroid = (UA && UA.indexOf('android') > 0) || (weexPlatform === 'android')
//是否IOS环境
export const isIOS = (UA && /iphone|ipad|ipod|ios/.test(UA)) || (weexPlatform === 'ios')
//是否Chrome环境
export const isChrome = UA && /chrome\/\d+/.test(UA) && !isEdge
//当前 window.navigator.userAgent 是否含有 phantomjs 关键字
export const isPhantomJS = UA && /phantomjs/.test(UA)
//是否火狐浏览器环境
export const isFF = UA && UA.match(/firefox\/(\d+)/)
```

```js
export const nativeWatch = ({}).watch
```
在火狐浏览器中原生提供了`Object.prototype.watch`方法。所以只有当前宿主环境为火狐的情况下，`nativeWatch`才会返回正确的函数，否则直接返回`undefined`。也可以用于区分`VUE`实例中的`watch`，防止冲突产生意外情况。
```js
//通过监听调用test-passive检测当前浏览器是否支持passive。
//其中代码用try catch 包裹起来，说明支持性较差，容易出错
export let supportsPassive = false
if (inBrowser) {
  try {
    const opts = {}
    Object.defineProperty(opts, 'passive', ({
      get () {
        supportsPassive = true
      }
    }: Object))
    window.addEventListener('test-passive', null, opts)
  } catch (e) {}
}

//检测是否服务端环境
let _isServer
export const isServerRendering = () => {
  if (_isServer === undefined) {
    if (!inBrowser && !inWeex && typeof global !== 'undefined') {
      _isServer = global['process'] && global['process'].env.VUE_ENV === 'server'
    } else {
      _isServer = false
    }
  }
  return _isServer
}

//检测是否开发工具环境
export const devtools = inBrowser && window.__VUE_DEVTOOLS_GLOBAL_HOOK__
```

```js
//检测当前方法是否是原生js提供的
export function isNative (Ctor: any): boolean {
  return typeof Ctor === 'function' && /native code/.test(Ctor.toString())
}

//检测当前宿主环境是否支持原生 symbol Reflect.ownKeys, 返回布尔值
export const hasSymbol =
  typeof Symbol !== 'undefined' && isNative(Symbol) &&
  typeof Reflect !== 'undefined' && isNative(Reflect.ownKeys)
```