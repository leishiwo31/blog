# JavaScript系列之——函数
## 一、什么是函数？
函数实际上也是对象，每个函数都是`Function`的实例，函数名就是指向函数的一个指针。因为函数也是对象，所以函数也拥有自己的属性和方法。
```js
function add(a,b){
  return a+b
}
add.reduce = function(a,b){
  return a-b
}
add(1,2);  // 2
add.reduce(2,1);  // 1
```
有的人会觉得函数用于自己的属性和方法很怪异，其实当函数拥有属性的时候，你可以认为函数就是一个`{}`对象即可，这样更加容易理解。
## 二、函数的执行过程
每当执行一个函数时，js解析器会先创建一个执行环境。具体可分为两个步骤：<br />
<b>步骤一：建立阶段</b>
- 建立活动对象
- 构建作用域链
- 确定this值
<b>步骤二：执行阶段</b>
- 变量赋值
- 执行其他代码
## 三、箭头函数与普通函数的区别
1、箭头函数在写法上一个很明显的地方是带有`=>`标志。且表达式更简洁。
```js
function add(a,b){
  return a + b
}
//用箭头函数写法是
function add((a,b) => a+b);
```
箭头函数与普通主要区别如下：
- 在创建箭头函数的时候，内部`this`的指向就已确定，与执行环境无关，且无法改变指向。
- 不可作为构造函数使用。
- 函数内部不可使用`arguments`对象，若要使用可使用`rest`参数代替。
- 函数内部也不可使用`super`、`new.target`。
- 箭头函数也没有`prototype`属性。
## 四、没有重载
在`java`语言中，一个函数可以有多个定义，只要接收到的参数的类型和数量不同就行，即签名不同即可。而在`javascript`中则不同，比如你定义的函数参数有5个，实际执行的时候你可以多传一些(多余的会被忽略)，或者少传一些都没问题，js解析器都不会报错。相同名称的函数在被第二次定义的时候，后者会覆盖前者。
## 五、默认参数
ES6开始，函数参数支持定义默认值。也就是说一个函数，定义的时候有3个参数，而每个都可以设置一个默认值，若执行的时候不传参数则按照默认值执行。
```js
function add(a=1,b=2){
  return a+b
}
add();  // 3
//而在ES6之前若定义默认参数，则一般会采用这种方法
function add(a,b){
  a = a || 1;
  b = b || 2;
  return a+b
}
//这种定义方法是有弊端的，比如你想执行 add(0,0)，得到的结果事与愿违。
//且ES5的定义方法比较麻烦，还需要判断各种异常，还不够直观
```
## 六、扩展运算符
如果你想定义一个函数，其参数数量无法估计，且无法预知的参数后面还需固定某个参数。这个时候`扩展运算符`的派上用场了。
```js
let arr = [2,3,4];
function add(){
 let result = 0;
 for(let i=0;i<arguments.length;i++){
   result += arguments[i]
 }
 return result
}
add(1,...arr,5);  
// 15
```
## 七、声明函数与函数表达式的区别
其实两者在写法上的区别非常明显
```js
// 声明函数:
function add(){}
// 函数表达式
var add = function(){}
```
两者在执行结果上没任何区别，但是`javascript`引擎在解析的时候，他们两者之间是区别对待的。前者在解析的时候，会有`变量提升`的效果，即将函数提升到当前作用域最开始部分，所以这样执行是不会有问题的：
```js
console.log(add(1,2));
function add(a,b){
  return a+b
}
```
而函数表达式则不会有这种`变量提升`的效果，所以这样会报错
```js
console.log(add(1,2));
var add = function(a,b){
  return a+b
}
```
## 八、关于`caller`、`callee`与`new.target`
`ECMAScript 5`会给每个声明的函数添加一个`caller`属性，指向调用当前函数的函数。全局函数的`caller`属性则为`null`。
```js
function outer(){
  inner();
}
function inner(){
  console.log(inner.caller)
};
outer();
// 打印出outer函数的源码
```
<br />函数内部可以调用`callee`。调用格式为`arguments.callee`，指向正在执行的函数的指针，一般为当前函数。注：严格模式下执行`arguments.callee`会报错。
```js
function add(){
  console.log(arguments.callee === add)
}
add();
// true
```
<br />`new.target`属性可以用来检测函数或者构造方法是否是通过`new`运算符调用的。如果是正常调用的，则`new.target`返回`undefined`，如果是使用`new`运算符调用的，则返回被调用的构造函数。
```js
function add(){
  if(new.target){
    console.log('add is using by "new"')
  }else{
    console.log('not used by "new"')
  }
}
new add();
// add is using by "new"
add();
// not used by "new"
```
另外一种方式判断函数是否通过`new`关键词调用(Vue源码中获取到的)：
```js
function add(){
  console.log(this instanceof add)
}
add();  // false
new add();  // true
```
## 九、立即调用函数
立即调用函数，简称`IIFE`，即声明即执行的一种函数表达式。早期`javascript`没有模块的概念，立即调用函数在一定程度上充当了`模块`的角色，函数内部的变量外部无法访问。
```js
var fn = (function(){
  var a = 1;
  console.log(a);
  // 1
})();
console.log(a);
// 报错
```
## 十、函数`length`、`name`属性
函数也拥有`长度`的概念，其值是定义的函数的参数的数量。
```js
function add(a,b){};
console.log(add.length);
// 2
```
<br />函数也拥有默认的`name`属性，返回该函数的函数名(字符串)。
```js
function add(){};
console.log(add.name === 'add');
// true
```
`Function`构造函数返回的函数实例，`name`属性的值为`anonymous`，翻译成中文就是“匿名的”。
```js
(new Function).name // "anonymous"
```
`bind`返回的函数，`name`属性值会加上`bound`前缀。
```js
function foo() {};
foo.bind({}).name // "bound foo"

(function(){}).bind({}).name // "bound "
```
## 十一、`Generator`函数
`Generator`函数是ES6提供的一种异步函数解决方案。特点：
- `function`关键字与函数名之间有一个星号；
- 函数内部可以使用`yield`表达式；
- 执行方式上不能像普通函数那样直接调用，需要调用后赋值给一个变量，执行变量的`next()`方法，返回值形式:`{value:"",done:false}`。其中`value`的值是`yield`表达式的返回值，`done`若为`true`则表示执行完毕，否则还可以继续执行变量的`next()`方法，直到`done`的值为`true`
```js
function *fn(){
  yield 1;
  yield 2;
  return 3
}
var c = fn();
c.next();  // {value:1,done:false}
c.next();  // {value:2,done:false}
c.next();  // {value:3,done:true}
c.next();  // {value:undefined,done:true}
```
当执行到函数尾部，或者遇到`return`的时候，返回的对象中的`done`的值则为`true`。每次执行`c.next()`的时候，都会一直执行，直到遇到`yield`表达式，则会`暂停`执行，直到下次继续执行`c.next()`。<br /><br />
### next()方法的参数<br />
支持传入一个参数值，在执行非第一次`next`方法的时候，若传入参数，则会被当做上一次`yield`的返回值，若不传则认为是`undefined`
```js
function *fn(x){
  var a = 2 * (yield (x+x));
  var b = 3 + (yield (a*2));
  return a+b
}
var r = fn(2);
r.next();  // {value:4,done:false}
r.next(6);  // {value:24,done:false}
r.next(3);  // {value:18,done:true}
```
有没有觉得有点绕？其实很简单，只要记住两点即可：1、`next()`遇到`yield`则暂停。2、`next`若带有参数，则可认为是上一个`yield`的返回值即可。分析下上面代码执行过程：<br />
- `r.next()`的时候，`x`的参数值是`2`，这个没毛病，然后执行第一段代码，从右向左执行，`yield`后的表达式是`x+x`，即`4`，然后遇到`yield`返回结果，此时并没有执行完，`a`的赋值操作被`暂停`了，此时返回`{value:4,done:false}`
- `r.next(6)`的时候，参数值`6`就可以认为是`yield (x+x)`的返回值，这个时候从上次中断的地方继续执行并赋值变量`a`的值为`12`，然后又遇到`yield (a*2)`得到的结果是`24`，于是又`暂停`执行，返回`{value:24,done:false}`
- `r.next(3)`的时候，这个时候就要执行`return a+b`这一段了。可以认为`yield (a*2)`这个整体就是`3`，则赋值变量`b`的值为`6`。最后`a+b`的值就是`18`，因为执行到`return`了，则整个方法执行完毕，于是返回`{value:18,done:true}`<br /><br />
### 思考一个问题：若定义一个函数，内部有N个`yield`，那我们岂不是得手动执行`next()`N次？
答案是肯定有解决方案的，不然与`javascript`宇宙第一语言的身份不相匹配啊。
```js
function *fn(){
    yield 1;
    yield 2;
    yield 3;
    return 4
}
for(let v of fn()){
  console.log(v)
}
// 1  2  3
```
注：遇到`done`返回值为`true`的时候，则会中断`for`循环。函数`return`的结果也不会在`for`循环中体现出来<br /><br />
### `Generator`函数的`return`方法
`Generator`函数返回的遍历器对象，除了支持`next()`方法调用外，还有一个方法`return`，主要能力是`阻断`后续`next()`的执行。比如函数中支持十个`yield`表达式，我们可以在中间位置`return`一下，`return`的结果就是`yield`的返回值，同时`done`返回`true`，则后续`next()`执行无效，即被认为函数已经执行完毕。
```js
function *fn(){
  yield 1;
  yield 2;
  yield 3;
  yield 4;
}
var c = fn();
c.next();   // {value:1,done:false}
c.return(123);  // {value:123,done:true}
c.next();  // {value:undefined,done:true}
```
注：若`return`无参数，则返回的`value`值为`undefined`
## 十二、`async`函数
`async`函数的基础是`Promise`，继续往下读的同学需要确保你对`Promise`有一定的了解。
`async`函数其实就是`Generator`函数的语法糖，其实都是为了如何将`异步`如何写的更像`同步`。特点：<br />
- `function`表达式前有一个`async`关键词，告诉js解析器本函数与普通函数不同。
- 函数内部支持`await`关键表达式，意思是告诉解析器`暂停`执行后续代码，等待后续返回结果后再继续。
`async`函数本身返回的也是一个`Promise`对象，支持`then`方法作为回调函数(`async`函数内部`return`的结果做为`then`的回调参数)，备注：只有`async`函数内部所有`await`执行完毕后才会回调`then`，或者函数内部`return`结果；也支持`catch`捕获函数内异常(若`async`函数内部报错，则会被`catch`捕获)。<br /><br />
### 关于 `await`
- 函数内部当遇到`await`的时候就会先返回执行其他代码，等`await`返回结果在从`await`位置继续执行后续操作;
- `await`命令后是一个`Promise`对象(也可以不是，若不是则直接返回对应值)，我们这里暂时成之为`newP`，若`newP resolve`结果后，则在之前停止的地方继续执行。若`reject`结果，则跳出当前`async`函数，则`async`函数的`catch`可以捕获到其中的异常；
### 多异步的`async`函数推荐写法
现实中我们可能会遇到一个`async`函数内部有多个`await`的情况，若他们之前相互没关联，建议同时执行，这样会加快程序的运行速度，否则一个执行完毕再执行另外一个，显然是不科学的。
```js
//例如有三个await的情况
async function fn(){
  let a = await fn1();
  let b = await fn2();
  let c = await fn3();
}
//推荐解构赋值写法
async function fn(){
  let [a,b,c] = Promise.all([fn1(),fn2(),fn3()]);
}
```

