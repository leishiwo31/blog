# Vue初始化之合并配置
构造函数`Vue`在生产环境的第一步就是执行原型链上的`_init`方法。该方法是在`initMixin`方法中定义，其中`options`就是我们调用`Vue`构造函数的时候传过来的，源代码位置:`/core/instance/init.js`
```js
export function initMixin (Vue: Class<Component>) {
  Vue.prototype._init = function (options?: Object) {
  	//初始化代码忽略
  }
}
```
## 合并选项之前都做了些啥
```js
const vm: Component = this
// a uid
vm._uid = uid++
```
首先声明`vm`常量指向当前`Vue`实例，然后给`vm`常量定义了一个内部变量`_uid`作为当前组件的唯一标识，每次初始化组件的时候，`_uid`依次递增。<br /><br />
```js
let startTag, endTag
/* istanbul ignore if */
if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
  startTag = `vue-perf-start:${vm._uid}`
  endTag = `vue-perf-end:${vm._uid}`
  mark(startTag)
}
```
接下来是性能测试。首先声明了两个变量`startTag`和`endTag`。如果是在非生产环境下，且`config.performance`和`mark`为`true`的情况下则执行性能追踪。<br /><br />
`config.performance`来自于`core/config.js`中的配置。在`Vue`官方文档中，我们看到可以对`config`进行修改配置。例如`Vue.config.performance = true`的时候，非生产环境主要对以下功能进行追踪：
+ 1、组件初始化(component init)
+ 2、编译(compile)，将模板(template)编译成渲染函数
+ 3、渲染(render)，其实就是渲染函数的性能，或者说渲染函数执行且生成虚拟DOM(vnode)的性能
+ 4、打补丁(patch)，将虚拟DOM渲染为真实DOM的性能<br /><br />
`mark`这里就不赘述了，在[工具方法篇](/vuejs/util/core/perf.html)中已做介绍
<br /><br />

```js
// a flag to avoid this being observed
vm._isVue = true
// merge options
if (options && options._isComponent) {
  // optimize internal component instantiation
  // since dynamic options merging is pretty slow, and none of the
  // internal component options needs special treatment.
  initInternalComponent(vm, options)
} else {
  //暂时忽略
}
```
首先在 `Vue` 实例上添加 `_isVue` 属性，并设置其值为 `true`。目的是用来标识一个对象是 `Vue` 实例，即如果发现一个对象拥有 `_isVue` 属性并且其值为 `true`，那么就代表该对象是 `Vue` 实例，这样可以避免该对象被响应系统观测。<br />
接下来是一个`if`...`else`分支。即如果选项上带有`_isComponent`内部选项，则表示这是已经初始过的组件，这里进行了一个优化策略(暂不做详细介绍，后面会再次讲到)。<br />
## 合并选项都做了些啥
接下来就是我们本篇文章的核心：<strong style="color:red;">选项合并策略</strong>。不少人对选项的合并往往嗤之以鼻，认为这不是响应式数据的核心，以为仅仅是两个对象合并成一个对象而已，没必要花太多心思去研究，所以不去深究。其实大错特错，深入研究后你才会发现不仅能从中学到不少东西，而且还可以深入了解每个组件的数据结构，这是理解`Vue`源码的基础。
```js
vm.$options = mergeOptions(
  resolveConstructorOptions(vm.constructor),
  options || {},
  vm
)
```
选项合并的目的就是将`Vue`默认配置与用户自定义的`options`进行合并后返回一个新的`Object`，并赋值给实例`vm`的属性`$options`。本质是将两个对象合并为一个新对象。<br /><br />
合并主要是`mergeOptions`函数做的事情，它接受三个参数：
+ 1、`resolveConstructorOption`函数返回的`Vue`默认配置
+ 2、用户自定义的默认配置。若无则传空对象
+ 3、当前实例本身

先来看下`resolveConstructorOption`函数，它的位置也在：`core/instance/index.js`
```js
export function resolveConstructorOptions (Ctor: Class<Component>) {
  let options = Ctor.options
  if (Ctor.super) {
    //暂时忽略
  }
  return options
}
```
`resolveConstructorOption`函数只接受一个参数`Ctor`，那么这个`Ctor`是啥呢？在实际调用的时候传的唯一参数是`vm.constructor`，正常情况下(注意是一般正常理解的情况下)这个值指向的其实就是`Vue`构造函数本身，举个例子：
```js
function testFn(){
  this.a = 1;
}
var vm = new testFn();
console.log(vm.constructor === testFn);  // true
```
所以`let options = Ctor.options`其实相当于`let options = Vue.options`。鉴于读者是刚看源码，对`Vue.extend`还不熟悉，为了便于理解，我们可以暂时这么理解:`resolveConstructorOptions(vm.constructor)`其实就是`Vue.options`。等整体看透之后再回头重看的时候，就能理解了。<br /><br />
不过这里还是会先简单介绍一下`resolveConstructorOptions`函数中`if`语句主要是干嘛的。
```js
if (Ctor.super) {
  const superOptions = resolveConstructorOptions(Ctor.super)
  const cachedSuperOptions = Ctor.superOptions
  if (superOptions !== cachedSuperOptions) {
    // super option changed,
    // need to resolve new options.
    Ctor.superOptions = superOptions
    // check if there are any late-modified/attached options (#4976)
    const modifiedOptions = resolveModifiedOptions(Ctor)
    // update base extend options
    if (modifiedOptions) {
      extend(Ctor.extendOptions, modifiedOptions)
    }
    options = Ctor.options = mergeOptions(superOptions, Ctor.extendOptions)
    if (options.name) {
      options.components[options.name] = Ctor
    }
  }
}
```
我们看到`if`语句的判断条件是`Ctor.super`，`super`是子类才有的属性。举个例子：
```js
var sub = Vue.extend();
var vm = new sub();
console.log(vm.constructor);  // sub
console.log(sub.super);  // Vue
```
以上例子也充分说明了，实例`vm`的`constructor`指向的不一定是`Vue`，这种情况指的是`sub`函数。因是继承而来的，所以`sub`函数有`super`属性。<br /><br />
<b>总结：目前只需要知道`resolveConstructorOptions`返回`Vue.options`即可，其中的`if(Ctor.super)`与`Vue.extend()`方法有关，等到后面遇到的时候再重点分析。</b>
### mergeOptions
接下来我们重点看下`mergeOptions`方法，它接受三个参数。<br />
第一个参数暂时认为是`Vue.options`：
```js
Vue.options = {
  components:{
    keepAlive,
    transition,
    transitinGroup
  },
  directives:{
    model,
    show
  },
  filters:Object.create(null),
  _base:Vue
}
```
第二个参数是我们调用`new Vue`的时候传的参数，例如：
```js
{
  el:'#app',
  data(){
    return {
      name:'wang'
    }
  },
  methods:{
    init(){

    }
  }
}
```
第三个参数就是当前实例本身`vm`。所以我们改下`mergeOptions`方法，其实就相当于：
```js
vm.$options = mergeOptions(
  {
    components:{
      keepAlive,
      transition,
      transitionGroup
    },
    directives:{
      model,
      show
    },
    filters:Object.create(null),
    _base:Vue
  },
  {
    el:'#app',
    data(){
      return {
        name:'wang'
      }
    },
    methods:{
      init(){

      }
    }
  },
  vm
)
```
接下来我们详细看下`mergeOptions`源代码。位置：`core/util/options.js`
```js
export function mergeOptions (
  parent: Object,
  child: Object,
  vm?: Component
): Object {
  if (process.env.NODE_ENV !== 'production') {
    checkComponents(child)
  }

  if (typeof child === 'function') {
    child = child.options
  }
  //剩余代码暂时忽略
}
```
首先在非生产环境下会使用`checkComponents`方法检测我们自定义配置中的组件名称命名是否规范。为何要在非生产环境检测呢？因为在非生产环境检测规范后，我们不大可能在build生产的时候再去修改代码，也就是说生产环境没必要再次检测，以此来达到节约性能的目的。类似的情况`process.env.NODE_ENV !== 'production'`以后会有很多，就不再一一赘述。检测代码如下：
```js
function checkComponents (options: Object) {
  for (const key in options.components) {
    validateComponentName(key)
  }
}
export function validateComponentName (name: string) {
  if (!new RegExp(`^[a-zA-Z][\\-\\.0-9_${unicodeRegExp.source}]*$`).test(name)) {
    warn(
      'Invalid component name: "' + name + '". Component names ' +
      'should conform to valid custom element name in html5 specification.'
    )
  }
  if (isBuiltInTag(name) || config.isReservedTag(name)) {
    warn(
      'Do not use built-in or reserved HTML elements as component ' +
      'id: ' + name
    )
  }
}
```
通过以上代码可以看出，检测的原理就是遍历我们传递选项的`components`属性。若满足以下条件，则打印警告：
+ 1、满足正则表达式`^[a-zA-Z][\\-\\.0-9_${unicodeRegExp.source}]*$`
+ 2、`isBuiltInTag(name) || config.isReservedTag(name)`之一成立的情况下

<b>说人话？好的好的</b><br />
其实第一条就是限定组件的命名规则由普通的字符和中横线(-)组成，且必须以字母开头。<br >
第二条是检测你的组件名称不能与内置标签(`slot`、`component`)冲突，也不能是内置标签。<br /><br />
说了这么多，其实就是保证在合并之前你的组件命名合理合法，作者为了防止开发者犯规，也是操碎了心呐。
### 允许我们传递的参数是一个函数
接下来的这段代码打破了我们上面说的：我们传递的合并参数是一个对象。其实它也可以是一个函数，这里增加了一个判断，若是函数，则`child`重新指向它的静态属性`child.options`。
```js
if (typeof child === 'function') {
  child = child.options
}
```
什么场景下会遇到这种情况呢？其实还是跟`Vue.extend()`函数有关，这个在后面也会详细讲解。
### 规范化props、inject、directives
### normalizeProps
```js
normalizeProps(child, vm)
normalizeInject(child, vm)
normalizeDirectives(child)
```
这三个函数是用来规范选项，方便后面合并而做的处理。为什么要这么做呢？以`props`为例，我们知道`Vue`允许以下多种写法：
```js
//写法一
const yourComponents = {
  props:['yourData']
}
//写法二
const yourComponents = {
  props:{
    yourData:{
      type:Number,
      default:1
    }
  }
}
//写法三
const yourComponents = {
  props:{
    yourData:{
      type:Number
    }
  }
}
//写法四
const yourComponents = {
  props:{
    yourData:Number
  }
}
```
这个给开发者提供了非常便利的选择，可以根据自己的习惯`任性`的写逻辑。但凡事都有两面性，开发者爽了，源码作者就要写更多方法来适应。也就是在真正合并之前，将开发者写的多种格式统一规范，方便后面合并。接下来以`normalizeProps`为例：
```js
function normalizeProps (options: Object, vm: ?Component) {
  const props = options.props
  if (!props) return
  const res = {}
  let i, val, name
  if (Array.isArray(props)) {
    i = props.length
    while (i--) {
      val = props[i]
      if (typeof val === 'string') {
        name = camelize(val)
        res[name] = { type: null }
      } else if (process.env.NODE_ENV !== 'production') {
        warn('props must be strings when using array syntax.')
      }
    }
  } else if (isPlainObject(props)) {
    for (const key in props) {
      val = props[key]
      name = camelize(key)
      res[name] = isPlainObject(val)
        ? val
        : { type: val }
    }
  } else if (process.env.NODE_ENV !== 'production') {
    warn(
      `Invalid value for option "props": expected an Array or an Object, ` +
      `but got ${toRawType(props)}.`,
      vm
    )
  }
  options.props = res
}
```
我们看到`normalizeProps`函数接受两个参数：第一个是开发者传递的`options`配置，第二个是可选的当前组件实例，主要用于非生产环境检测异常(`props`格式非数组也非对象)后打印警告。
```js
function normalizeProps (options: Object, vm: ?Component) {
  const props = options.props
  if (!props) return
  const res = {}
  let i, val, name
  //暂时忽略此处代码
  options.props = res
}
```
我们将代码缩减下看看，首先读取当前开发者配置中的`props`，若不存在则直接返回。之后声明了新对象`res`，用于保存规范后的结果输出，同时又声明了`i`、`val`、`name`三个变量供后面使用。
```js
function normalizeProps (options: Object, vm: ?Component) {
  //暂时忽略
  if (Array.isArray(props)) {
    i = props.length
    while (i--) {
      val = props[i]
      if (typeof val === 'string') {
        name = camelize(val)
        res[name] = { type: null }
      } else if (process.env.NODE_ENV !== 'production') {
        warn('props must be strings when using array syntax.')
      }
    }
  } else if (isPlainObject(props)) {
    for (const key in props) {
      val = props[key]
      name = camelize(key)
      res[name] = isPlainObject(val)
        ? val
        : { type: val }
    }
  } else if (process.env.NODE_ENV !== 'production') {
    warn(
      `Invalid value for option "props": expected an Array or an Object, ` +
      `but got ${toRawType(props)}.`,
      vm
    )
  }
  options.props = res
}
```
我们可以看到，这里执行了三个`if`分支判断。其中用到了`camelize`方法，可移步查看[camelize](/vuejs/util/shared/util.html#camelize)
+ 若是数组格式，`while`循环遍历数组的每个元素。只有元素是字符串格式的时候，改成`{type:null}`格式。例如开发者写的是`props:['yourData']`，转换后的结果就是`props:{yourData:{type:null}}`。若元素非字符串且非生产环境下则打印警告。
+ 若是纯对象格式，`for`循环遍历对象的每个元素。先保存`value`结果，在规范命名规则，之后判断若`val`是纯对象，则直接使用，否则改成`{type:val}`的格式。比如
```js
// 例子一：
props:{
  yourData:Number
}
// 将会被修改为以下格式
props:{
  yourData:{
    type:Number
  }
}


//例子二：
props:{
  yourData:{
    type:Number
  }
}
// 不做格式转变，直接使用。至于是否包含默认值，对转换结果没影响
```
<b>总结：其实规范化`props`很简单，只是将数据修改为纯对象格式。对象增加一个`type`属性，若有指定类型则显示类型，否则为`null`。若有默认值`default`也会包含在里面。</b>
### normalizeInject
详细了解了如何规范化`props`之后，再看另外两个规范想必就非常容易了，这里就简单介绍了：
```js
function normalizeInject (options: Object, vm: ?Component) {
  const inject = options.inject
  if (!inject) return
  const normalized = options.inject = {}
  if (Array.isArray(inject)) {
    for (let i = 0; i < inject.length; i++) {
      normalized[inject[i]] = { from: inject[i] }
    }
  } else if (isPlainObject(inject)) {
    for (const key in inject) {
      const val = inject[key]
      normalized[key] = isPlainObject(val)
        ? extend({ from: key }, val)
        : { from: val }
    }
  } else if (process.env.NODE_ENV !== 'production') {
    warn(
      `Invalid value for option "inject": expected an Array or an Object, ` +
      `but got ${toRawType(inject)}.`,
      vm
    )
  }
}
```
这里有一点非常有意思：先定义变量`inject`缓存`options.inject`的值，之后定义`normalized`与`options.inject`同时指向一个空对象。其中用到了`extend`方法，可移步查看[`extend`](/vuejs/util/shared/util.html#extend)
+ 若是数组格式，则遍历改成`{yourKey:{from:yourKey}}`格式；
+ 若是纯对象，若`val`是纯对象，则改成`{yourKey:{from:yourKey,yourData:yourData}}`格式；否则还是`{yourKey:{from:yourKey}}`格式。
### normalizeDirectives
规范化`directives`
```js
function normalizeDirectives (options: Object) {
  const dirs = options.directives
  if (dirs) {
    for (const key in dirs) {
      const def = dirs[key]
      if (typeof def === 'function') {
        dirs[key] = { bind: def, update: def }
      }
    }
  }
}
```
当且仅当`directives`存在且每一项的`value`值是函数的时候，数据修改前后如下：
```js
// 修改前
{
  directives:{
    a:function(){}
  }
}
//修改后
{
  directives:{
    a:{
      bind:function(){},
      update:function(){}
    }
  }
}
```
#### 总结：规范化数据格式至此告一段落，它们的存在只是为后面真正的合并做一个规范化处理，保持数据格式统一，开发者的命名规范。
### mixins、extends的合并方式
我们知道`mixins`用于解决代码复用的问题。接下来的这段代码就是将开发者配置中可能存在的`mixins`混入合并到`parent`中。
```js
if (!child._base) {
  if (child.extends) {
    parent = mergeOptions(parent, child.extends, vm)
  }
  if (child.mixins) {
    for (let i = 0, l = child.mixins.length; i < l; i++) {
      parent = mergeOptions(parent, child.mixins[i], vm)
    }
  }
}
```
首先判断`!child._base`的情况下才会执行上面代码。而`child`是开发者传递的配置，默认肯定是没有的，那肯定会执行里面的内容。那么什么情况下才会有`child._base`呢？我们知道`_base`只有`Vue.config`默认配置中有这么一个属性，且指向的是构造函数`Vue`本身，而合并是取两个对象的最大值为一个新对象，所以合并后的`vm.$options._base`结果肯定是有的了。所以结论是：只有第一次合并，是原始合并选项，不是另一次`mergeOptions`的结果再合并的时候，才会执行这里的代码。这么做是防止重复执行，一方面是节省性能，另一方面也是没必要。
+ 若开发者配置中存在`mixins`，则遍历`mixins`中的每个元素，递归调用`mergeOptions`方法合并产生一个新的对象，并赋值给`parent`；
+ 而若开发者配置中存在`extends`，则更简单，因为`extends`只是一个对象，相当于`mixins`的一个元素，直接递归调用即可。

### 主要选项的合并方式
```js
const options = {}
let key
for (key in parent) {
  mergeField(key)
}
for (key in child) {
  if (!hasOwn(parent, key)) {
    mergeField(key)
  }
}
function mergeField (key) {
  const strat = strats[key] || defaultStrat
  options[key] = strat(parent[key], child[key], vm, key)
}
```
做了这么多铺垫，现在才轮到真正合并的阶段。合并的原理并不复杂：<br />
+ 首先创建一个用于最终输出结果的`options`对象；
+ `for`循环遍历`parent`合并到`options`中；
+ `for`循环遍历`child`，只有`parent`中没有的`key`对象才能合并到`options`中(防止开发者覆盖默认配置)；
+ `mergeField`函数是核心合并方式。在这之前先定义了`strats`<b>策略对象</b>，对象上分别定义了`el`、`propsData`、`data`、`生命周期`、`components`、`directives`、`filters`、`watch`、`props`、`methods`、`inject`、`computed`、`provide`的合并方式，称为<b style="color:red;">选项合并策略</b>，即不同的模块采用不同的合并方式；
+ 若找不到指定合并方式，例如开发者定义了一个特殊的选项：`child.aabbcc = {}`，这个时候`strats.aabbcc`的结果是`undefined`，这种情况则会调用默认合并方法`defaultStrat`。
+ 默认合并方法是在没指定合并策略的前提下使用的，若开发者定义了一个特殊的选项`child.aabbcc={}`，我们也可以提前在全局配置下定义<b>同名</b>合并方法：`Vue.config.optionMergeStrategies.aabbcc = function(){}`。这就是`Vue`文档上提到的[自定义选项合并策略](https://cn.vuejs.org/v2/guide/mixins.html#%E8%87%AA%E5%AE%9A%E4%B9%89%E9%80%89%E9%A1%B9%E5%90%88%E5%B9%B6%E7%AD%96%E7%95%A5)

接下来我们来一一查看各个模块的合并方法。
### el、propsData 合并策略
```js
if (process.env.NODE_ENV !== 'production') {
  strats.el = strats.propsData = function (parent, child, vm, key) {
    if (!vm) {
      warn(
        `option "${key}" can only be used during instance ` +
        'creation with the `new` keyword.'
      )
    }
    return defaultStrat(parent, child)
  }
}
```
在非生产环境定义了`strats.el`、`strats.propsData`，其最终调用的还是默认合并方法`defaultStrat`。这里有人就要奇怪了，为何只在非生产环境定义，生产环境怎么办？其实生产环境也是调用的默认合并方法`defaultStrat`，因为`strats.el`、`strats.propsData`的结果是`undefined`，没有的话就会采用默认合并策略。<br />
其实`Vue`中无论哪个环境，其最终输出结果必定是一致的。如果实现的过程有区别，那一定是为了方便开发调试，这里唯一的区别是多了一个`if(!vm){}`判断没有`vm`实例的情况下打印警告，提示`el`选项或者`propsData`选项只能在使用`new`操作符创建实例的时候可用。这也说明了，如果拿不到`vm`则说明处理的是子组件选项。<br />
### defaultStrat 默认合并策略
```js
const defaultStrat = function (parentVal: any, childVal: any): any {
  return childVal === undefined
    ? parentVal
    : childVal
}
```
接下来就是策略合并的默认方法`defaultStrat`。当一个选项不需要特殊处理的时候，就使用默认合并策略。逻辑很简单：若`childVal`存在则直接返回，否则返回`parentVal`。
### data 合并策略
```js
strats.data = function (
  parentVal: any,
  childVal: any,
  vm?: Component
): ?Function {
  if (!vm) {
    if (childVal && typeof childVal !== 'function') {
      process.env.NODE_ENV !== 'production' && warn(
        'The "data" option should be a function ' +
        'that returns a per-instance value in component ' +
        'definitions.',
        vm
      )

      return parentVal
    }
    return mergeDataOrFn(parentVal, childVal)
  }

  return mergeDataOrFn(parentVal, childVal, vm)
}
```
`data`的合并策略代码如上所示，其最终主要使用`mergeDataOrFn`方法处理，有三种可能性：
+ 如果是子组件选项，且开发者所写的`data`不是函数的情况下(`data`必须是函数)则不做合并处理，直接返回`parentVal`结果；
+ 如果是子组件选项，且开发者所传`data`格式合规(是函数)的情况下，直接调用`mergeDataOrFn`方法处理`data`结果；
+ `vm`存在的情况下，也就是当`new Vue`的时候(因为这个时候`vm`值是必然存在的)，也直接调用`mergeDataOrFn`方法处理`data`结果。

<b>总结：子组件与根组件合并`data`选项都是调用了`mergeDataOrFn`方法处理，唯一的区别是是否传`vm`参数。</b><br /><br />
那么`mergeDataOrFn`方法到底是怎么处理的呢？我们先来看下源码：
```js
export function mergeDataOrFn (
  parentVal: any,
  childVal: any,
  vm?: Component
): ?Function {
  if (!vm) {
    // in a Vue.extend merge, both should be functions
    if (!childVal) {
      return parentVal
    }
    if (!parentVal) {
      return childVal
    }
    // when parentVal & childVal are both present,
    // we need to return a function that returns the
    // merged result of both functions... no need to
    // check if parentVal is a function here because
    // it has to be a function to pass previous merges.
    return function mergedDataFn () {
      return mergeData(
        typeof childVal === 'function' ? childVal.call(this, this) : childVal,
        typeof parentVal === 'function' ? parentVal.call(this, this) : parentVal
      )
    }
  } else {
    return function mergedInstanceDataFn () {
      // instance merge
      const instanceData = typeof childVal === 'function'
        ? childVal.call(vm, vm)
        : childVal
      const defaultData = typeof parentVal === 'function'
        ? parentVal.call(vm, vm)
        : parentVal
      if (instanceData) {
        return mergeData(instanceData, defaultData)
      } else {
        return defaultData
      }
    }
  }
}
```
通过上面代码我们可以看出`mergeDataOrFn`分为两种情况：<br /><br />
<b>情况一：不存在`vm`属性的时候，说明处理的是子组件选项。</b>
+ 根据注释可以得知：当前是调用`Vue.extend`函数时进行合并处理的，要求此时处理的父子`data`都必须是函数类型。
+ 接下来是两个`if`判断。如果没有`childVal`，说明子组件选项中没有`data`选项，则直接返回父组件选项。同样的，若负组件中不存在`data`选项，则无需合并，直接返回子组件`data`选项。
+ 当`parentVal`、`childVal`都存在的情况下才会真正执行`data`的合并策略，直接返回一个`mergedDataFn`方法，此时`data`的合并代码直接执行结束，返回的是一个函数`mergeDataFn`。所以：`data`原本是一个函数，合并后仍然是一个函数，而不是一个纯对象。
+ `mergeDataFn`方法内部返回的是函数`mergeData`执行后返回的结果。而`mergeData`的两个参数则是子父`data`方法执行后返回的纯对象格式。

<b>情况二：存在`vm`属性的时候，说明处理的是非子组件选项，也就是处理`new`操作符创建实例的情况。</b>
+ 此时也是返回的一个未执行的函数`mergedInstanceDataFn`
+ 执行`childVal`与`parentVal`方法分别得到对应的纯对象`instanceData`与`defaultData`。如果子类`data`方法存在，则直接调用`mergeData`方法合并两个对象为一个纯对象作为`mergedInstanceDataFn`函数的返回值，若不存在则直接返回父类对象作为`mergedInstanceDataFn`函数的返回值。

<b>总结：`mergeDataOrFn`方法无论是否有`vm`属性，最终返回的永远是一个未执行的函数。内部只是调用了`mergeData`方法，将父子`data`函数执行后得到的纯对象合并之后得到合并后的纯对象。</b>
<br /><br /><br />
上面提到了`mergeData`方法，那么这个方法是如何工作的呢？我们继续看它的源码：
```js
function mergeData (to: Object, from: ?Object): Object {
  if (!from) return to
  let key, toVal, fromVal

  const keys = hasSymbol
    ? Reflect.ownKeys(from)
    : Object.keys(from)

  for (let i = 0; i < keys.length; i++) {
    key = keys[i]
    // in case the object is already observed...
    if (key === '__ob__') continue
    toVal = to[key]
    fromVal = from[key]
    if (!hasOwn(to, key)) {
      set(to, key, fromVal)
    } else if (
      toVal !== fromVal &&
      isPlainObject(toVal) &&
      isPlainObject(fromVal)
    ) {
      mergeData(toVal, fromVal)
    }
  }
  return to
}
```
此方法接受两个参数`to`、`from`，后者非必需，若不存在，则直接返回`to`。从`mergeDataOrFn`函数中`mergeData`执行时的传参顺序看，`to`相当于`childVal`函数返回的对象，`from`相当于`parentVal`函数返回的对象。`mergeData`方法的作用是遍历`from`对象数据合并到`to`上，最终返回`to`对象。知道整体逻辑后，我们再详细拆解：
+ 若`from`不存在，那就没有合并的必要了，直接返回`to`；
+ 声明三个未赋值的变量`key`、`toVal`、`fromVal`；
+ 获取`from`对象的`key`值组成的数组并赋值给`keys`。至于怎么获取，这里做了个判断，若宿主环境支持原生`symbol`、`Reflect`，则使用`Reflect.ownKeys`获取，否则使用`Object.keys`方法获取；
+ 遍历对象，如果发现`key`值是`__ob__`，则跳过继续执行下一个循环。`__obj__`是啥？是响应式观测数据，后面会详细讲到，这里只要知道`__ob__`属性不会被合并即可；
+ 如果`from`对象中的`key`值不存在`to`中，则调用`set`函数对`to`设置对应的值；
+ 如果`from`对象中的`key`值存在`to`中，且`from[key]`与`to[key]`不全等，且两者都是纯对象的情况下，则递归调用`mergeData`深度合并。

`mergeData`函数中用到了`set`函数，根据引用路径得知这个函数的位置：`core/observer/index.js`。里面的逻辑较多，后面在讲到响应式的时候会详细解释，目前我们只提取当前用到的代码，方便大家理解：
```js
export function set (target: Array<any> | Object, key: any, val: any): any {
  //暂时忽略
  const ob = (target: any).__ob__
  //暂时忽略
  if (!ob) {
    target[key] = val
    return val
  }
  //暂时忽略
}
```
### 生命周期选项合并策略
源码中`strats.data...`合并策略之后就是生命周期选项的合并策略，源码如下：
```js
// Hooks props 最终都会被合并为数组格式
function mergeHook (
  parentVal: ?Array<Function>,
  childVal: ?Function | ?Array<Function>
): ?Array<Function> {
  const res = childVal
    ? parentVal
      ? parentVal.concat(childVal)
      : Array.isArray(childVal)
        ? childVal
        : [childVal]
    : parentVal
  return res
    ? dedupeHooks(res)
    : res
}

function dedupeHooks (hooks) {
  const res = []
  for (let i = 0; i < hooks.length; i++) {
    if (res.indexOf(hooks[i]) === -1) {
      res.push(hooks[i])
    }
  }
  return res
}

LIFECYCLE_HOOKS.forEach(hook => {
  strats[hook] = mergeHook
})

//以下代码来自  src/shared/constants.js
export const LIFECYCLE_HOOKS = [
  'beforeCreate',
  'created',
  'beforeMount',
  'mounted',
  'beforeUpdate',
  'updated',
  'beforeDestroy',
  'destroyed',
  'activated',
  'deactivated',
  'errorCaptured',
  'serverPrefetch'
]
```
从上面代码可以很容易看出：遍历`LIFECYCLE_HOOKS`数组，将生命周期钩子函数每一项挂到`strats`策略对象上，全部指向`mergeHook`函数。这说明合并生命周期选项的核心就是`mergeHook`函数，整个函数体由<b>三组三目</b>运算符组成。我们接下来拆解看看`mergeHook`函数都做了些啥：
<br />
+ `mergeHook`函数接受两个参数，第一个是父类生命周期钩子，第二个是子类生命周期钩子；
+ `childVal`、`parentVal`都存在的情况下，则`res`是他们合并后的数组结果。注意：`parentVal`在前；
+ `childVal`存在，`parentVal`不存在的情况下，这个时候若`childVal`是数组则赋值给`res`，否则将`childVal`作为唯一元素组成数组后赋值给`res`；
+ 若`childVal`不存在，则直接将`parentVal`赋值给`res`；
+ `mergeHook`函数最后返回一个数组，若`res`为`true`，即存在的情况下，则返回去重后的数组；

学习了生命周期合并原则后，我们发现了一个新的好玩的东西：生命周期不仅仅可以写成一个函数，还可以写成函数组成的数组格式。
```js
// 平时我们会这么写
{
  //代码忽略
  created:function(){
    console.log('created')
  }
  //代码忽略
}

//其实也可以这么写
{
  //代码忽略
  created:[
    function(){
      console.log('created1')
    },
    function(){
      console.log('created2')
    }
  ]
  //代码忽略
}
```
<b>总结：生命周期合并最终都会被合并成一个数组的格式，他们并不会被相互替换。会本着父辈在前，子类在后的原则。实际执行的时候，也是本着这个原则。</b>

### assets 选项合并策略
```js
function mergeAssets (
  parentVal: ?Object,
  childVal: ?Object,
  vm?: Component,
  key: string
): Object {
  const res = Object.create(parentVal || null)
  if (childVal) {
    process.env.NODE_ENV !== 'production' && assertObjectType(key, childVal, vm)
    return extend(res, childVal)
  } else {
    return res
  }
}

ASSET_TYPES.forEach(function (type) {
  strats[type + 's'] = mergeAssets
})
//以下代码来自  src/shared/constants.js
export const ASSET_TYPES = [
  'component',
  'directive',
  'filter'
]
```
通过`ASSET_TYPES`的内容我们可以看到，`Vue`中`components`、`directives`、`filters`被认为是资源。与生命周期合并原则类似，遍历`ASSET_TYPES`分别在`strats`上指定合并策略方法是`mergeAssets`。此方法的逻辑也很简单：
+ 创建一个原型为`parentVal`的对象`res`；
+ 若子类不存在，则直接返回`res`；
+ 若子类存在，则直接将`childVal`遍历复制到`res`中，最后直接返回`res`。

其中，非生产环境调用了`assertObjectType`方法，源码如下：
```js
function assertObjectType (name: string, value: any, vm: ?Component) {
  if (!isPlainObject(value)) {
    warn(
      `Invalid value for option "${name}": expected an Object, ` +
      `but got ${toRawType(value)}.`,
      vm
    )
  }
}
```
其目的就是在非生产环境下检测`childVal`，确保其为纯对象，若不是则给出警告。

<b>总结：静态资源的合并就是先创建一个父辈为原型的空对象，将子类合并到空对象后返回</b>

### watch 合并策略
顺着源码位置继续往下，`assets`选项合并后就是`watch`选项合并，源代码主要如下：
```js
strats.watch = function (
  parentVal: ?Object,
  childVal: ?Object,
  vm?: Component,
  key: string
): ?Object {
  // work around Firefox's Object.prototype.watch...
  if (parentVal === nativeWatch) parentVal = undefined
  if (childVal === nativeWatch) childVal = undefined
  /* istanbul ignore if */
  if (!childVal) return Object.create(parentVal || null)
  if (process.env.NODE_ENV !== 'production') {
    assertObjectType(key, childVal, vm)
  }
  if (!parentVal) return childVal
  const ret = {}
  extend(ret, parentVal)
  for (const key in childVal) {
    let parent = ret[key]
    const child = childVal[key]
    if (parent && !Array.isArray(parent)) {
      parent = [parent]
    }
    ret[key] = parent
      ? parent.concat(child)
      : Array.isArray(child) ? child : [child]
  }
  return ret
}
```
+ 首先确保父子都不是火狐浏览器对象原型链上的`watch`，若是，则置空。
+ 若子类不存在，则直接返回原型为父类的空对象；
+ 接下来是在非生产环境下检测子类是否是纯对象，若不是则给出警告；
+ 如果父类不存在，则直接返回子类；
+ 接下来就是父子都存在的情况下的合并。首先将父类遍历合并到空对象`ret`上。接下来就是遍历子类每一项，检测父类是否包含同名选项，若有则需确保父类同名选项为数组格式。
+ 最后将子类每一项`复制`到`ret`对象中，若与父类名称冲突则返回与父类合并后的新数组，若不冲突则返回当前选项元素组成的新数组。
<b>总结：合并后的`watch`选项，若父子存在同名，则同名元素的值为数组格式，否则还是一个函数</b>

### props、methods、inject、computed 合并策略
`watch`合并源代码后是这四个家伙的合并策略，期源代码逻辑如下：
```js
strats.props =
strats.methods =
strats.inject =
strats.computed = function (
  parentVal: ?Object,
  childVal: ?Object,
  vm?: Component,
  key: string
): ?Object {
  if (childVal && process.env.NODE_ENV !== 'production') {
    assertObjectType(key, childVal, vm)
  }
  if (!parentVal) return childVal
  const ret = Object.create(null)
  extend(ret, parentVal)
  if (childVal) extend(ret, childVal)
  return ret
}
```
合并原理很简单：
+ 首先是非生产环境检测子类是否是纯对象，若不是则打印警告；
+ 若父类没有，则直接返回子类；
+ 创建一个没有原型的空对象，将父类对应的内容遍历拷贝到空对象`ret`中；
+ 若子类也存在，则将子类内容遍历拷贝到`ret`中；
+ 最后返回拷贝后的对象`ret`。

<b>总结：这四个选项的合并原则很简单，就是创建一个没有原型的纯对象，遍历拷贝父子类到新对象即可。</b>

### provide 合并策略
```js
strats.provide = mergeDataOrFn
```
最后就是`provide`的合并策略，合并方法就是`mergeDataOrFn`。前面已经讲过，这里不再赘述。