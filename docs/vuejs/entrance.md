# 从Vue构造函数入口开始
我们知道`Vue`往往是从`new`开始工作的，这就说明`Vue`一定是个构造函数，那么它的原型链上肯定会暴露很多方法或者属性。鉴于`Vue`的规则，在看源码之前先大胆猜测一下，以下写法或者类似写法，应该会非常多：
```js
Vue.prototype.$method = function(){
	//忽略
}
Vue.prototype._method = function(){
	//忽略
}
```
只是具体怎么实现的，或者说`Vue`的神秘之处有哪些，看完之后会不会拍大腿，我们带着这个疑问一探究竟。从入口位置顺藤摸瓜：`core/instance/index.js`
```js
import { initMixin } from './init'
import { stateMixin } from './state'
import { renderMixin } from './render'
import { eventsMixin } from './events'
import { lifecycleMixin } from './lifecycle'
import { warn } from '../util/index'

function Vue (options) {
  if (process.env.NODE_ENV !== 'production' &&
    !(this instanceof Vue)
  ) {
    warn('Vue is a constructor and should be called with the `new` keyword')
  }
  this._init(options)
}

initMixin(Vue)
stateMixin(Vue)
eventsMixin(Vue)
lifecycleMixin(Vue)
renderMixin(Vue)

export default Vue

```
我们看到一开始页面引入了五大`Mixin`方法，执行的时候唯一的参数就是`Vue`构造函数本身。主要是在不同阶段给`Vue`原型链上添加了不同的方法，本篇会先简单介绍这五大方法分别暴露了哪些方法或者属性，后面会有详细文章介绍。先来看下构造函数本身：
```js
function Vue (options) {
  if (process.env.NODE_ENV !== 'production' &&
    !(this instanceof Vue)
  ) {
    warn('Vue is a constructor and should be called with the `new` keyword')
  }
  this._init(options)
}
```
构造函数主要做两件事：
+ 在非生产环境检测调用`Vue`是否是`new`出来的，如若不是则会打印一个警告；
+ 将你的`options`配置传入，执行`Vue`原型链上的内部方法`_init`。
### initMixin
主要做了哪些事情：
+ 合并配置
+ 初始化组件实例关系以及内部属性：比如当前组件的父组件($parent)、根组件($root)、子组件数组($children)容器、$refs以及内部属性等。
+ 初始化自定义事件
+ 初始化组件插槽信息
+ 调用生命周期函数`beforeCreate`
+ 初始化组件的 inject 配置项
+ 初始化状态。分别初始化 `props`、`methods`、`data`、`computed`、`watch`(<strong style="color:red;">划重点！！！</strong>)
+ 初始化组件的 provide 配置项
+ 调用生命周期函数`create`
+ 如果是根组件(自定义配置上有`el`选项)则调用`$mount`自动挂载
### stateMixin
构造函数`Vue`原型链上初始化`$data`、`$props`属性，初始化`$set`、`$delete`、`$watch`方法
### eventsMixin
构造函数`Vue`原型链上初始化`$on`、`$once`、`$off`、`$emit`方法
### lifecycleMixin
构造函数`Vue`原型链上初始化`_update`、`$forceUpdate`、`$destroy`方法
### renderMixin
构造函数`Vue`原型链上初始化`$nextTick`、`_render`方法
### 总结
`Vue`将主逻辑分别封装在不同的`Mixin`方法中初始化，写的非常清楚，更方便了定位分析，非常值得我们学习借鉴。<br />
本篇只梳理了`Vue`初始化主要做了哪些事情的结构，顺着这个主心骨，接下来再一一详细解剖。