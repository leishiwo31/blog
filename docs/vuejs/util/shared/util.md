### shared/util.js
> 正所谓磨刀不误砍柴工，在真正阅读源码之前先了解清楚一些变量是如何判断、纯函数的作用是十分有必要的。知道锤子、斧头、扳手、剪刀是干嘛用的，这样才不至于在初次见面的时候显得措手不及。

### emptyObject
```js
export const emptyObject = Object.freeze({})
```
创建一个冻结的空对象，这就意味着emptyObject不可扩展，也就是不能添加新的属性

### isUndef
```js
export function isUndef (v: any): boolean %checks {
  return v === undefined || v === null
}
```
判断变量是否未定义，也就是判断是否是定义了未赋值，或者赋值null的情况

### isDef
```js
export function isDef (v: any): boolean %checks {
  return v !== undefined && v !== null
}
```
判断变量是否定义

### isTrue
```js
export function isTrue (v: any): boolean %checks {
  return v === true
}
```
判断变量是否为true

### isFalse
```js
export function isFalse (v: any): boolean %checks {
  return v === false
}
```
判断变量是否为false

### isPrimitive
```js
export function isPrimitive (value: any): boolean %checks {
  return (
    typeof value === 'string' ||
    typeof value === 'number' ||
    typeof value === 'symbol' ||
    typeof value === 'boolean'
  )
}
```
判断变量是否是原始值，也就是判断变量是否未非复合型数据

### isObject
```js
export function isObject (obj: mixed): boolean %checks {
  return obj !== null && typeof obj === 'object'
}
```
判断变量是否是 “对象” 或者 “数组”

### toRawType
```js
const _toString = Object.prototype.toString
export function toRawType (value: any): string {
  return _toString.call(value).slice(8, -1)
}
```
判断类型后返回原始字符串类型。例如判断一个数组:`Object.prototype.toString.call([])`得到的结果是`[object Array]`，再使用`slice(8,-1)`后得到的结果就是`Array`字符串

### isPlainObject
```js
export function isPlainObject (obj: any): boolean {
  return _toString.call(obj) === '[object Object]'
}
```
判断变量是否是纯对象

### isRegExp
```js
export function isRegExp (v: any): boolean {
  return _toString.call(v) === '[object RegExp]'
}
```
判断变量是否是正则对象

### isValidArrayIndex
```js
export function isValidArrayIndex (val: any): boolean {
  const n = parseFloat(String(val))
  return n >= 0 && Math.floor(n) === n && isFinite(val)
}
```
判断变量是否是有效的数组索引。
+ 正常的数组索引格式应该是大于等于0的整数
+ 先将变量解析并返回一个浮点数，保存在变量n中
+ 当且仅当变量大于等于0，且是整数，并且是一个有限数值的情况下才会返回true
### isPromise
```js
export function isPromise (val: any): boolean {
  return (
    isDef(val) &&
    typeof val.then === 'function' &&
    typeof val.catch === 'function'
  )
}
```
判断一个变量是否是Promise


### toString
```js
export function toString (val: any): string {
  return val == null
    ? ''
    : Array.isArray(val) || (isPlainObject(val) && val.toString === _toString)
      ? JSON.stringify(val, null, 2)
      : String(val)
}
```
将变量转换成字符串格式
+ 若是null，则返回空字符串
+ 若是数组或纯对象，则使用JSON.stringify处理
+ 其他情况直接String函数处理

### toNumber
```js
export function toNumber (val: string): number | string {
  const n = parseFloat(val)
  return isNaN(n) ? val : n
}
```
将变量转换成数字类型，若转换失败(比如转换字符串"aaa"得到的结果是NaN)则返回初始值

### makeMap
```js
export function makeMap (
  str: string,
  expectsLowerCase?: boolean
): (key: string) => true | void {
  const map = Object.create(null)
  const list: Array<string> = str.split(',')
  for (let i = 0; i < list.length; i++) {
    map[list[i]] = true
  }
  return expectsLowerCase
    ? val => map[val.toLowerCase()]
    : val => map[val]
}
```
返回一个函数，判断给定的字符串中是否包含指定内容(映射)。<br />
> str 数据格式： 以`,`间隔的字符串，例如:`a,b,c,d,e` <br />
> `expectsLowerCase`：是否将`map`的值`key`小写 <br />

使用示例(检测是否是小写的元音字母)：
```js
let isLower = makeMap('a,b,c,d,e');
isLower('b');  // true
isLower('f');  // undefined
```
### remove
```js
export function remove (arr: Array<any>, item: any): Array<any> | void {
  if (arr.length) {
    const index = arr.indexOf(item)
    if (index > -1) {
      return arr.splice(index, 1)
    }
  }
}
```
从数组中删除给定元素，并返回被删除项组成的数组：`[item]`
> 在数组不为空的情况下，获取当前元素序号，当元素存在的前提下(大于-1)，则使用`splice`方法删除，并返回被删除的元素组成的新数组，否则返回`undefined`

使用示例：
```js
let arr = ['a','b','c'];
remove(arr,'a');  // ['a']
//arr变成 ['b','c']
```
### hasOwn
```js
// 封装`Object.prototype.hasOwnProperty`，用于检测给定对象中是否含有给定`key`值
const hasOwnProperty = Object.prototype.hasOwnProperty
export function hasOwn (obj: Object | Array<*>, key: string): boolean {
  return hasOwnProperty.call(obj, key)
}
```
### cached
```js
export function cached<F: Function> (fn: F): F {
  const cache = Object.create(null)
  return (function cachedFn (str: string) {
    const hit = cache[str]
    return hit || (cache[str] = fn(str))
  }: any)
}
```
为一个纯函数创建创建一个缓存版本的函数。输入的参数必须是一个纯函数，得到的返回也是一个纯函数。
+ 首先我们要确认为何一定要传一个纯函数，因为纯函数的输出结果只与输入有关，所运行的环境不能改变输出值。
+ 创建一个原型链为空的闭包对象`cache`用以缓存结果。
+ 随后返回一个函数 `cachedFn`，优先读取缓存，如果有则直接返回返回的值，否则使用原函数`fn`计算一次并缓存结果。

### camelize
```js
const camelizeRE = /-(\w)/g
export const camelize = cached((str: string): string => {
  return str.replace(camelizeRE, (_, c) => c ? c.toUpperCase() : '')
})
```
将中横线连字符转换成驼峰式命名

### capitalize
```js
export const capitalize = cached((str: string): string => {
  return str.charAt(0).toUpperCase() + str.slice(1)
})
```
将字符串首字母改成大写

### hyphenate
```js
const hyphenateRE = /\B([A-Z])/g
export const hyphenate = cached((str: string): string => {
  return str.replace(hyphenateRE, '-$1').toLowerCase()
})
```
与`camelize`方法相反，将驼峰式命名改成连字符方式。例如：`hyphenateRE('aaBb')` => `aa-bb`

### polyfillBind
### nativeBind
### bind
```js
function polyfillBind (fn: Function, ctx: Object): Function {
  function boundFn (a) {
    const l = arguments.length
    return l
      ? l > 1
        ? fn.apply(ctx, arguments)
        : fn.call(ctx, a)
      : fn.call(ctx)
  }

  boundFn._length = fn.length
  return boundFn
}

function nativeBind (fn: Function, ctx: Object): Function {
  return fn.bind(ctx)
}

export const bind = Function.prototype.bind
  ? nativeBind
  : polyfillBind
```
以上其实就是一个`bind`函数，绑定`this`指向后返回的一个新函数。<br />
+ `polyfillBind`为手动写的一个绑定函数
+ `nativeBind` Function原型链上的原生绑定函数
+ `bind` 优先判断是否支持原生绑定函数，若支持则优先使用原生，否则使用手动实现的函数


### toArray
```js
export function toArray (list: any, start?: number): Array<any> {
  start = start || 0
  let i = list.length - start
  const ret: Array<any> = new Array(i)
  while (i--) {
    ret[i] = list[i + start]
  }
  return ret
}
```
从指定位置(默认从0开始)开始将类数组转换成数组的方法。
+ `list` 类数组
+ `start`可选的开始索引位置

### extend
```js
export function extend (to: Object, _from: ?Object): Object {
  for (const key in _from) {
    to[key] = _from[key]
  }
  return to
}
```
将一个对象(`_from`)的数据拷贝到另一个对象(`to`)上。若`key`值有重复，则`_from`中的会替换`to`中的数据。注意：若`value`值为复合型数据，则是浅拷贝。

### toObject
```js
export function toObject (arr: Array<any>): Object {
  const res = {}
  for (let i = 0; i < arr.length; i++) {
    if (arr[i]) {
      extend(res, arr[i])
    }
  }
  return res
}
```
遍历数组每个对象元素，将内容拷贝到一个新对象上，并返回新对象。for循环遍历数组，与`extend`函数一起使用，拷贝到新对象`res`中，逻辑很简单。

### noop
```js
export function noop (a?: any, b?: any, c?: any) {}
```
一个空函数，什么都不做。

### no
```js
export const no = (a?: any, b?: any, c?: any) => false
```
始终返回`false`的函数。

### identity
```js
export const identity = (_: any) => _
```
将输入值直接返回的纯函数

### genStaticKeys
```js
export function genStaticKeys (modules: Array<ModuleOptions>): string {
  return modules.reduce((keys, m) => {
    return keys.concat(m.staticKeys || [])
  }, []).join(',')
}
```
从编译器模块生成包含静态键的字符串。
+ `modules`是一个数组，是编译器的一个选项，其中每个元素都是一个可能包含`staticKeys`属性的对象。此方法的目的就是遍历数组元素，收集`staticKeys`以`,`隔开组成的拼接字符串。

### looseEqual
```js
export function looseEqual (a: any, b: any): boolean {
  if (a === b) return true
  const isObjectA = isObject(a)
  const isObjectB = isObject(b)
  if (isObjectA && isObjectB) {
    try {
      const isArrayA = Array.isArray(a)
      const isArrayB = Array.isArray(b)
      if (isArrayA && isArrayB) {
        return a.length === b.length && a.every((e, i) => {
          return looseEqual(e, b[i])
        })
      } else if (a instanceof Date && b instanceof Date) {
        return a.getTime() === b.getTime()
      } else if (!isArrayA && !isArrayB) {
        const keysA = Object.keys(a)
        const keysB = Object.keys(b)
        return keysA.length === keysB.length && keysA.every(key => {
          return looseEqual(a[key], b[key])
        })
      } else {
        /* istanbul ignore next */
        return false
      }
    } catch (e) {
      /* istanbul ignore next */
      return false
    }
  } else if (!isObjectA && !isObjectB) {
    return String(a) === String(b)
  } else {
    return false
  }
}
```
判断两个值是否全等。

### looseIndexOf
```js
export function looseIndexOf (arr: Array<mixed>, val: mixed): number {
  for (let i = 0; i < arr.length; i++) {
    if (looseEqual(arr[i], val)) return i
  }
  return -1
}
```
查找给定元素是否在指定数组中，若存在则返回当前元素所在数组中的索引，否则返回-1。

### once
```js
export function once (fn: Function): Function {
  let called = false
  return function () {
    if (!called) {
      called = true
      fn.apply(this, arguments)
    }
  }
}
```
利用闭包的特性实现一个只调用一次的函数。











