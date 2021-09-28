### lang.js
> 正所谓磨刀不误砍柴工，在真正阅读源码之前先了解清楚一些变量是如何判断、纯函数的作用是十分有必要的。知道锤子、斧头、扳手、剪刀是干嘛用的，这样才不至于在初次见面的时候显得措手不及。

### 本文件只暴露一个变量，三个方法
```js
export const unicodeRegExp = /a-zA-Z\u00B7\u00C0-\u00D6\u00D8-\u00F6\u00F8-\u037D\u037F-\u1FFF\u200C-\u200D\u203F-\u2040\u2070-\u218F\u2C00-\u2FEF\u3001-\uD7FF\uF900-\uFDCF\uFDF0-\uFFFD/
```
unicode正则表达式，用于解析template模板<br /><br />

```js
export function isReserved (str: string): boolean {
  const c = (str + '').charCodeAt(0)
  return c === 0x24 || c === 0x5F
}
```
检测字符串是否是以`_`或者美元符号`$`开头，主要用来检测VUE组件实例中变量的命名是否符合规范(VUE不建议以`_`或`$`为开头的字符串命名变量，这是框架保留命的名方式)。<br /><br />
```js
export function def (obj: Object, key: string, val: any, enumerable?: boolean) {
  Object.defineProperty(obj, key, {
    value: val,
    enumerable: !!enumerable,
    writable: true,
    configurable: true
  })
}
```
应该是为了方便调用，作者将`Object.defineProperty`做了一个简单的封装。`def`函数支持四个参数，分别是源对象本身、key值、value值、是否可枚举，最后一个参数`enumerable`非必填，默认不可枚举。<br /><br />
```js
const bailRE = new RegExp(`[^${unicodeRegExp.source}.$_\\d]`)
export function parsePath (path: string): any {
  if (bailRE.test(path)) {
    return
  }
  const segments = path.split('.')
  return function (obj) {
    for (let i = 0; i < segments.length; i++) {
      if (!obj) return
      obj = obj[segments[i]]
    }
    return obj
  }
}
```
对路径进行一个简单的解析，单看这个方法其实比较简单：若`path`符合正则(例如含有`~`、`/`、`*`)则返回(其实是边界处理，确保不传入异常值而已)，否则用`.`切割`path`成一个数组，保存在变量`segments`中，随后返回一个函数，函数内遍历访问`segments`得到一个对象。<b>此方法主要用于初始化`watcher`的时候触发值的读取。</b>