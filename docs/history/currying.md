# 小白都能看懂的函数柯里化
JavaScript函数柯里化与反柯里化是高阶函数应用之一，那么什么是高阶函数？其实只要将函数当做参数传递的情况，就是高阶函数。比如日常开发中大家都遇到过的回调函数，这些都是高阶函数。那么问题来了，什么是函数柯里化？`函数柯里化是一种将使用多个参数的函数转换成一系列的使用单个参数的技术`。举个简单的栗子：
```js
function add(a,b,c){
  return a+b+c
}
//柯里化之后
function addCurrying(a){
  return function(b){
    return a+b
  }
}
//原函数调用方式
add(1,2);
//柯里化的调用方式
addCurrying(1)(2);
```
实际上`函数柯里化`就是把add函数的x、y参数变成了先用函数接收x然后返回一个函数再去处理y参数。<br /><br />
思考：如果参数数量未知，显然上面的情况不具备通用性，无法满足实际需要。例如:`addCurrying(1,2)(2,3,4,5,6)(3)(4)`，那么我们就考虑一下如何封装，适配任意参数的情况。显然有横向和纵向两个特点，横向是可以执行无数个方法，纵向是参数个数不确定。那么接着面的例子，我们继续。
```js
//先解决纵向的情况：可传任意数量参数
function addCurrying(){
  let _args = Array.from(arguments);
  return function(){
    _args.push(...arguments);
    return _args.reduce((pre,cur) => pre+cur,0)
  }
}
//执行下结果
addCurrying(1)(2);  // 3
addCurrying(1,2)(3,4,5);  // 15
```
以上方法用到的知识点:<br />
- `arguments`表示函数内部参数组成的类数组(注意：箭头函数内无`arguments`)
- `Array.from`将类数组转换为真正的数组，方便后面累加计算。转换方法很多，还有`Array.of(...arguments)`或者`Array.prototype.slice.call(arguments)`等
- `闭包`，将参数统统存放在`_args`数组内
- `reduce`累加得到结果
上面例子解决了纵向问题，那么横向怎么办呢？对，没错，递归可以搞定，因为横向是无数个，所以一定要确保`return`的结果一定是`function`，稍加改造：
```js
function addCurrying(){
  let _args = Array.from(arguments);
  let adapter = function(){
    //将每次执行方法的参数保存在闭包变量中
    _args.push(...arguments);
    return adapter
  }
  return adapter
}
```
OK，至此解决了横向无限循环调用的问题。那么问题来了，如何实现累加呢？答案就是利用`toString`隐式转换特性，在最后执行时隐式转换，计算并返回累加值
```js
function addCurrying(){
  let _args = Array.from(arguments);
  let adapter = function(){
    //将每次执行方法的参数保存在闭包变量中
    _args.push(...arguments);
    return adapter
  }
  adapter.toString = function(){
    return _args.reduce((pre,cur) => pre+cur,0)
  }
  return adapter
}
```
test一下试试看，结果完美！
```js
addCurrying(1,2,3)(4);  // f 10
addCurrying(1,2,3)(4)(5,6);  // f 21
addCurrying(1,2,3)(4)(5,6)(7,8,9);  // f 45
```