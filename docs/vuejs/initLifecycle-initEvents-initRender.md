## Vue初始化之initLifecycle
再次回到`_init`函数中，上一篇我们花了大量的时间详细描述了`vm.$options`是如何策略合并的。接下来我们顺着代码继续往下看:
```js
if (process.env.NODE_ENV !== 'production') {
  initProxy(vm)
} else {
  vm._renderProxy = vm
}
vm._self = vm
```
我们看到在非生产环境执行`initProxy(vm)`，而生产环境则直接定义`_renderProxy`属性指向当前实例本身。本着生产环境和非生产环境功能必须保持一致的原则，`initProxy(vm)`的目的，其实也是在实例上增加一个`_renderProxy`属性。鉴于是非生产环境，这里不再展开讨论，只用一句话总结`initProxy`的作用：通过原生`Proxy`对实例对象`vm`进行代理。<br />
那么 `_renderProxy`到底是干嘛用的呢？目前只是定义个指向，是后面`vm.$options.render`函数中`this`的指向。<br /><br />
<b>接下来我们回归本篇正题：`initLifecycle`都做了些啥</b><br />
```js
export function initLifecycle (vm: Component) {
  const options = vm.$options

  // locate first non-abstract parent
  let parent = options.parent
  if (parent && !options.abstract) {
    while (parent.$options.abstract && parent.$parent) {
      parent = parent.$parent
    }
    parent.$children.push(vm)
  }

  vm.$parent = parent
  vm.$root = parent ? parent.$root : vm

  vm.$children = []
  vm.$refs = {}

  vm._watcher = null
  vm._inactive = null
  vm._directInactive = false
  vm._isMounted = false
  vm._isDestroyed = false
  vm._isBeingDestroyed = false
}
```
源代码很少，逻辑也相对简单：
+ 定义一个常量`options`指向`vm.$options`；
+ 搜索当前实例的父实例，如果没有父实例，则给当前实例的`$parent`设置为`undefined`，比如当前组件是`new`出来的，那么就是根组件，父辈肯定是`undefined`了；
+ 如果当前`parent`存在且不是抽象组件，则将当前实例添加到父组件的`$children`中，成为他的`孩子`之一；
+ 找到当前组件的根组件`$root`，其实就是父组件的根组件。为何这样就认为是根组件了呢？因为组件是树状关系，每一个“出生”的孩子在出生的时候就认准了自己的祖先，这样代代相传，每生一个孩子就告诉他：孩子，你的祖先是谁谁谁，可不能忘本啊。这样千秋万代，都知道自己的祖先叫啥了。
+ 接下来就是定义子组件容器`$children`、`$refs`等，都是与`生命周期`有关的数据，这里先占个坑位。

那么问题来了，什么叫“抽象组件”？一个最大的区别就是它不渲染真实DOM，例如`keep-alive`、`transiton`、`transition-group`，抽象组件实例中一定有个属性`abstract:true`。
## Vue初始化之initEvents
初始化完毕`initLifecycle`之后，紧接着就是初始化`initEvents`。老规矩，先上源码：
```js
export function initEvents (vm: Component) {
  vm._events = Object.create(null)
  vm._hasHookEvent = false
  // init parent attached events
  const listeners = vm.$options._parentListeners
  if (listeners) {
    updateComponentListeners(vm, listeners)
  }
}
```
初始化自定义事件，开始在实例上定义了一个内部原型为`null`的空对象，然后又增加一个内部属性`_hasHookEvent`，初始值为`false`。之后是判断`$options`上是否含有`_parentListeners`，若存在则执行`updateComponentListeners`方法。这里就有点奇怪，`$options`上哪来的`_parentListeners`属性呢？其实根组件初始化的时候是没有的，这是初始化子组件需要执行的，这里暂时先忽略，后面会详细讲解。
## Vue初始化之initRender
`initEvents`简单带过后，我们接着看下一个初始化的函数`initRender`，老规矩，先上源码：
```js
export function initRender (vm: Component) {
  vm._vnode = null // the root of the child tree
  vm._staticTrees = null // v-once cached trees
  const options = vm.$options
  const parentVnode = vm.$vnode = options._parentVnode // the placeholder node in parent tree
  const renderContext = parentVnode && parentVnode.context
  vm.$slots = resolveSlots(options._renderChildren, renderContext)
  vm.$scopedSlots = emptyObject
  // bind the createElement fn to this instance
  // so that we get proper render context inside it.
  // args order: tag, data, children, normalizationType, alwaysNormalize
  // internal version is used by render functions compiled from templates
  vm._c = (a, b, c, d) => createElement(vm, a, b, c, d, false)
  // normalization is always applied for the public version, used in
  // user-written render functions.
  vm.$createElement = (a, b, c, d) => createElement(vm, a, b, c, d, true)

  // $attrs & $listeners are exposed for easier HOC creation.
  // they need to be reactive so that HOCs using them are always updated
  const parentData = parentVnode && parentVnode.data

  /* istanbul ignore else */
  if (process.env.NODE_ENV !== 'production') {
    defineReactive(vm, '$attrs', parentData && parentData.attrs || emptyObject, () => {
      !isUpdatingChildComponent && warn(`$attrs is readonly.`, vm)
    }, true)
    defineReactive(vm, '$listeners', options._parentListeners || emptyObject, () => {
      !isUpdatingChildComponent && warn(`$listeners is readonly.`, vm)
    }, true)
  } else {
    defineReactive(vm, '$attrs', parentData && parentData.attrs || emptyObject, null, true)
    defineReactive(vm, '$listeners', options._parentListeners || emptyObject, null, true)
  }
}
```
首先定义两个初始值都为`null`的属性`_vnode`、`_staticTrees`，先占个位，混个眼熟，功能后面自然会用到。剩下的代码相对复杂，目前不一一解释，只简单总结：初始化插槽信息`$slots`以及初始化`$createElement`方法，使用`defineReactive`方法让`$attrs`、`$listeners`响应式。


