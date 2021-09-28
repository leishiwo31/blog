# Vue生命周期钩子函数实现原理
## 执行生命周期钩子函数beforeCreate
```js
callHook(vm, 'beforeCreate')
```
讲了这么久，终于到了生命周期函数这一节。其主要逻辑都集中在`callHook`方法中，我们先来看下这个函数里都有些啥，源码位置：`src/core/instance/lifecycle.js`
```js
export function callHook (vm: Component, hook: string) {
  pushTarget()
  const handlers = vm.$options[hook]
  const info = `${hook} hook`
  if (handlers) {
    for (let i = 0, j = handlers.length; i < j; i++) {
      invokeWithErrorHandling(handlers[i], vm, null, vm, info)
    }
  }
  if (vm._hasHookEvent) {
    vm.$emit('hook:' + hook)
  }
  popTarget()
}
```
首先在讲解原理之前，我们看到`callHook`函数头尾分别执行了`pushTarget()`、`popTarget()`方法，这么做是为了防止在执行生命周期钩子函数中重复定义响应式数据。然后再回顾一下之前`$options`合并一节中生命周期策略合并方法，最终生命周期钩子函数合并后的结果一定是数组格式。
+ 从当前实例配置中找到生命周期钩子函数同名数组；
+ 若存在，则遍历当前数组中的方法，依次执行；
+ `invokeWithErrorHandling`函数可能会让大家有点迷惑，其实就是函数的`call`或者`apply`额外封装了一些异常情况，若有异常则直接打印问题。
接下来看下`invokeWithErrorHandling`函数：
```js
export function invokeWithErrorHandling (
  handler: Function,
  context: any,
  args: null | any[],
  vm: any,
  info: string
) {
  let res
  try {
    res = args ? handler.apply(context, args) : handler.call(context)
    if (res && !res._isVue && isPromise(res) && !res._handled) {
      res.catch(e => handleError(e, vm, info + ` (Promise/async)`))
      // issue #9511
      // avoid catch triggering multiple times when nested calls
      res._handled = true
    }
  } catch (e) {
    handleError(e, vm, info)
  }
  return res
}
```
我们看到首先用`try..catch...`包裹起来，若出现异常则直接打印。根据第三个参数`args`是否数组，选择使用`call`还是`apply`调用执行函数。若传进来的第一个参数`handler`是`Promise`类型，增加`catch`捕获异常并打印出来。
<br /><br />
<b>一句话总结：生命周期钩子执行就是遍历钩子函数数组，依次执行其中的方法。</b>