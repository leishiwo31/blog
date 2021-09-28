### perf.js
> 正所谓磨刀不误砍柴工，在真正阅读源码之前先了解清楚一些变量是如何判断、纯函数的作用是十分有必要的。知道锤子、斧头、扳手、剪刀是干嘛用的，这样才不至于在初次见面的时候显得措手不及。

### 性能监控专用 <br />
定义非生产环境用于监控性能的两个变量`mark`、`measure`，生产环境无需监控，直接返回`undefined`。源代码如下：
```js
import { inBrowser } from './env'

export let mark
export let measure

if (process.env.NODE_ENV !== 'production') {
  const perf = inBrowser && window.performance
  /* istanbul ignore if */
  if (
    perf &&
    perf.mark &&
    perf.measure &&
    perf.clearMarks &&
    perf.clearMeasures
  ) {
    mark = tag => perf.mark(tag)
    measure = (name, startTag, endTag) => {
      perf.measure(name, startTag, endTag)
      perf.clearMarks(startTag)
      perf.clearMarks(endTag)
      // perf.clearMeasures(name)
    }
  }
}
```
当且仅当宿主是浏览器且在非生产环境下，`perf`就是`window.performance`。if语句中做了多个判断，就是为了确保当前环境支持`window.performance`。之后重新定义`mark`、`measure`，以上代码在非生产环境下简化后的结果就是：
```js
const mark = tag => window.performance.mark(tag);
const measure = (name,startTag,endTag) => {
	window.performance.measure(name,startTag,endTag)
	window.performance.clearMarks(startTag)
	window.performance.clearMarks(endTag)
}
```
### window.performance是什么？
Web Performance API允许网页访问某些函数来测量网页和Web应用程序的性能。详细介绍请移步[window.performance](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/performance)