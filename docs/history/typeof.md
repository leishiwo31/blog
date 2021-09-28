# 重学JavaScript系列之——语言基础
笔者最近对原生JS知识做一个梳理，会将整个过程贴出来，内容细节尽量涵盖所有js知识点，同时会有一个不断进阶的过程。毕竟js是前端er的根本，重学再多遍也不为过。本系列对JS初学者来说将会有很大收获，同时对中级开发者也会有很好的提升，高级开发者也会得到复习和巩固，建议收藏后不定期回看。
## 一、数据类型
基本上数据类型7个(最后2个为ES6新增)：<br />
- string
- number
- boolean
- null
- undefined
- symbol
- bigint
<b>引用数据类型分为“基本引用类型”和“集合引用类型”：</b><br />
基本上引用类型：
- Date
- RegExp
- Math
集合引用类型：
- Object
- Array
- Function
- Set
- WeakSet
- Map
- WeakMap
<b>原始类型</b>是最简单的数据，<b>引用类型</b>则是由多个值构成的对象。在把一个值赋值给变量的时候，js引擎需要先确认这个值是原始值还是引用值，因为实际操作的是存储在变量中的实际值。引用值是保存在内存中的对象，声明一个对象给变量的时候，实际是将这个变量指向对象，而且可能存在多个变量指向同一个对象的情况。当改变其中一个值的时候，你会发现另外一个变量访问这个值的时候数据也发生了变化，例如：`var obj = {a:1};var obj2 = obj;obj.a = 2;`这个时候你会发现`obj2.a`的值变也变成了2。搞懂这一点对理解集合引用类型非常重要，后面会详细介绍。
## 二、变量与声明
js支持`var、let、const、function、import`声明或引入变量关键词。<br />
### 1、var声明
在ES6之前，声明原始数据采用`var`关键词。有以下一些特点
```js
//* 声明作用域 *//
function msg(){
  var a = 1; //局部变量
}
msg();
console.log(a);  //报错
```
```js
//* 声明提升 *//
function msg(){
  console.log(a);
  if(false){
   var a = 1;
  }
}
msg();  // undefined
```
执行以上代码并不会报错，因为js在执行之前会先扫描当前代码，将声明的变量自动提升到当前作用域顶部。以上方法相当于
```js
function msg(){
  var a;
  console.log(a);
  if(false){
   a = 1;
  }
}
```
注：同时存在变量提升的还有function声明的方法。<br />
### 2、let声明
let与var作用差不多，但有着非常明显的区别，let具有块状作用域，非let声明范围内无法访问。不存在变量提升，若未声明就试图访问会报错，这就是“暂时性死区”。特点：<br />
- 不存在变量提示
- 暂时性死区
- 同一个作用域范围内不允许重复声明

```js
if(true){
  console.log(a);  //报错
  let a = 1;
  let a = 2;  //报错
}
console.log(a);  //Uncaught ReferenceError: a is not defined
```
### 3、const声明
const的行为与let基本相同，还有如下特性：<br />
- 声明一个只读的常量，一旦声明，常量的值就不能改变。
- 一旦声明就必须赋值，否则会报错；
注：const声明的限制只适用于它指向的变量的引用。也就是说如果const变量引用的是一个对象或者数组等复合型数据，改变数据内部属性并不违反规定。
## 三、数据类型的判断
### 1、万能判断方法
```js
function getType(va){
  return Object.prototype.toString.call(val)
}
//判断后的类型大约有以下内容
var typeList = [
  "[object String]",
  "[object Number]",
  "[object Null]",
  "[object Undefined]",
  "[object Boolean]",
  "[object Symbol]",
  "[object BigInt]",
  "[object HTMLHtmlElement]",
  "[object Arguments]",
  "[object Math]",
  "[object Date]",
  "[object Object]",
  "[object Array]",
  "[object Set]",
  "[object WeakSet]",
  "[object Map]",
  "[object WeakMap]",
]
```
### 2、typeof判断方法
基本数据类型：除了`null`以外，用typeof 可以判断。
```js
typeof 2;  // number
typeof 'a';  //string
typeof false;  // boolean
typeof undefined;  // undefined
typeof Symbol('1');  // symbol
typeof BigInt(10);  // bigint
```
复合型：除 `Function`外，其他均返回`object`；<br />
```js
typeof [];  // object
typeof new Object(); // object
```
那么`typeof`是不是就不能用来判断数据类型了呢？答案是可以用，但是要用其他组合。
```js
//判断是否对象
function getType(val){
  return typeof val === 'object' && val.constructor === Object
}
//判断是否数组
function getType(val){
  return typeof val === 'object' && val.constructor === Array
}
/* 其他复合型数据以此类推 */
```
在排除对象的前提下，也可以采用`instanceof`方法判断。`instanceof`的原理是基于原型链的查询，只要处于原型链中，就会返回`true`。但要注意一点：`Object`是所有原型链的顶层，这点容易被忽略
```js
[] instanceof Array;  // true
[] instanceof Object; // true
```
也就是说当定义的一个变量`val instanceof Array`返回`true`的时候，不一定就能说明`val`就是数组！
### 3、Array.isArray()
原生js内置方法，专门用来判断是否为数组
```js
var arr = [];
Array.isArray(arr);  // true
```
### 4、如何手动实现一个`instanceof`方法
`instanceof`方法的原理是顺着原型链往上查找，直到`null`为止。
```js
function myInstanceof(from,to){
  //若是null或者基本上数据类型，则直接返回 false
  if(from === null || typeof from !== 'object') return false;
  var _proto = Object.getPrototypeOf(from);  //返回实例原型的原型链
  while(_proto){
    //查找到最顶层
    if(_proto === null) return false;
    if(_proto === to.prototype) return true
    _proto = Object.getPrototypeOf(_proto);
  }
}
```
### 5、`null`为什么不是对象？
`typeof`判断一个变量是否为`object`的时候，只有`null`是个例外。因为`null`明明是基本数据类型，却没有像我们期待的那样返回`null`。为何会这样呢？因为：这是js的一个bug导致的。虽然`typeof null`返回的结果是`object`，但它也不是对象。JS在最初版本中使用32位系统，`null`转换为计算机可识别的二进制时全为`0`，而对象恰好是`000`开头，所以将`null`也错误的认为是`object`了。
## 四、操作符与语句
### 1、一元操作符
只操作一个值的操作符叫做<b>一元操作符</b><br />
一元操作符有`++`和`--`两种，分别有前置和后置的区别。前置`++`是先自加再使用，后置`++`是先使用再自加。`--`操作符亦如此。
```js
var num = 0;
console.log(num++);  //打印结果为 0，这个时候num变为 1
console.log(++num);  //打印结果为 2，这个时候num变为 2
console.log(--num);  //打印结果为 1，num值变为 1
console.log(num--);  //打印结果为 1，num值变为 0
```
### 2、加性操作符
加性操作符，即加法操作符和减法操作符。在js中，当两个数值相加或相减的时候，会发生不同数据类型的转换。
```js
1+1; // 2
"1"+1;  // "11"
[]+{};  // "[object Object]"
[]+1;  // "1"
{}+1;  //"1"
new Set()+1;  // "[object Set]1"
NaN + NaN;  // NaN
"1"+NaN;  // "1[NaN]"
```
如果有一个操作数是字符串，则要应用如下规则：
- 如果两个操作数都是字符串，则将第二个字符串拼接到第一个字符串后面；
- 如果只有一个操作数是字符串，则将另一个操作数转换为字符串，再将两个字符串拼接在一起。
如果有任一操作数是对象、数值或布尔值，则调用它们的 toString()方法以获取字符串，然后再
应用前面的关于字符串的规则。对于 undefined 和 null，则调用 String()函数，分别获取
"undefined"和"null"。<br />
———— 摘自《JavaScript高级程序设计第四版》
### 3、相等操作符
主要分为相等`==`和全等`===`，反之为`!=`和`!===`。前者会进行类型转换后再做比较，后者除检验数值相等之外，还需确保类型相等。<br />
在转换操作数的类型时，相等和不相等操作符遵循如下规则。<br />
- 如果任一操作数是布尔值，则将其转换为数值再比较是否相等。false 转换为 0，true 转换为1。
- 如果一个操作数是字符串，另一个操作数是数值，则尝试将字符串转换为数值，再比较是否相等。
- 如果一个操作数是对象，另一个操作数不是，则调用对象的 valueOf()方法取得其原始值，再根据前面的规则进行比较。
在进行比较时，这两个操作符会遵循如下规则。<br />
- null 和 undefined 相等。
- null 和 undefined 不能转换为其他类型的值再进行比较。
- 如果有任一操作数是 NaN，则相等操作符返回 false，不相等操作符返回 true。记住：即使两个操作数都是 NaN，相等操作符也返回 false，因为按照规则，NaN 不等于 NaN。 
- 如果两个操作数都是对象，则比较它们是不是同一个对象。如果两个操作数都指向同一个对象，则相等操作符返回 true。否则，两者不相等。
```js
null == undefined;  // true
"NaN" == NaN;       // false
5 == NaN;           // false
NaN == NaN;         // false
NaN != NaN;         // true
false == 0;         // true
true == 1;          // true
true == 2;          // false
undefined == 0;     // false
null == 0;          // false
"5" == 5;           // true
```
———— 摘自《JavaScript高级程序设计第四版》<br /><br />
<b>对象类型相等运算转换规则</b><br/>
若相等运算符其中一个值为对象类型，则会按照以下顺序执行，直到转换成原始类型。
- 如果存在`Symbol.toPrimitive()`方法，则优先调用此方法；
- 调用`valueOf()`方法，若转换成原始数值类型，则返回结果；
- 调用`toString()`方法尝试转换成原始数值类型；
- 若都没返回原始数值类型，则报错；
<b>全等操作符`===`和不全等操作符`!==`</b><br />
它们与相等操作符类似，只不过它们在对比的时候不进行操作符类型转换，也就是说前后类型也必须相等才会返回`true`。注：JS规定，任何时候`NaN`均不相等。
```js
NaN === NaN;   // false
NaN ==  NaN;   // false
```
<b>`Object.is()`</b><br />
鉴于相等运算符`==`和全等运算符`===`都有自己的缺点。前者会进行类型的隐式转换，后者`NaN`不等于自身，以及`+0`等于`-0`。在ES6中推出了`Object.is`方法，用于比较两个运算符是否相等，其支持两个参数。
```js
Object.is({},{});    // false
Object.is(NaN,NaN);  // true
Object.is(+0,-0);    // false
```
### 4、循环语句
<b>1.`do{}while()`与`while`语句</b><br />
两者类似，只是写法的区别。使用的时候注意`while`条件，不要写死循环，防止内存溢出。
```js
var i = 0;
do{
  console.log(i);
  i++
}while(i < 10);
//打印 0 - 9的数值

while (i<20){
  i++;
  console.log(i)
}
```
<b>2.for循环</b><br />
这里主要介绍如何退出for循环体系。主要有两个`continue`与`break`，前者用于立即强制执行下一个循环，后者用于立即退出当前for循环
```js
for(var i=0;i<5;i++){
  if(i === 3) continue;
  console.log(i)
}
//打印结果  0  1  2  4

for(var i=0;i<5;i++){
  if(i===3) break;
  console.log(i)
}
//打印结果  0  1  2
```
<b>3.`for in`与`for of`语句</b><br />
`for in`:主要用于获取键名，遍历所有可枚举的属性；支持遍历字符串、数组、对象等。<br />
`for of`:主要用于获取健值，支持遍历字符串、数组、Set、Map等。前提是遍历的对象已经部署原生的iterator接口，否则会报错。特别注意**不支持遍历对象**，比如遍历`{a:1,b:2}`会报错<br />
```js
var obj = {a:1,b:2};
var arr = ['a','b'];
```
```js
for(var key in obj){
  console.log(key)
}

//打印结果  ['a','b']
for(var val of obj){
  console.log(val)
}
//打印结果  Uncaught TypeError: obj is not iterable

for(var key in arr){
  console.log(key)
}
//打印结果  0  1

for(var val of arr){
  console.log(val)
}
//打印结果  'a' 'b'
```
注意点：<br />
- `for in`在遍历对象的时候，会检查原型链，若要准确判断当前对象是否含有对应属性，可用`Object.hasOwnProperty`属性；
- 获取对象属性的时候，也可使用`Object.keys()`方法；
- `for in`或者`for of`均无法获取到Symbol类型数据，因为Symbol不可枚举，若要判断Symbol可使用`Object.getOwnPropertySymbols()`方法；
<b>4.`switch`语句</b><br />
## 五、作用域与内存
### 1、函数中参数的传递
ECMAScript中所有函数的参数都是按值传递的。也就是说传到函数内部的参数，都会复制一份。若参数为基本数据类型，则不会影响函数外变量。
```js
var num = 0;
function add(s){
  s += 1;
  console.log(s)
}
add(num);  // s的结果为 1
console.log(num);  // num的值仍然为0
```
若函数参数为复合型数据，则有可能改变原数据
```js
var persion = {
  name:"wang",
  sex:"man"
}
function set(val){
  val.name = "li";
  val = {
    name:"zhang",
    sex:"women"
  }
  return val
}
var s = set(persion);
console.log(persion);  // {name:"li",sex:"man"}
console.log(s);  // {name:"zhang",sex:"women"}
```
为什么会这样呢？因为`set`方法执行的时候，传入的参数是`persion`，函数中第一句话会改变函数外部`persion`属性`name`的值。第二句执行的时候，`val`变量指向一个新创建的对象，函数随后返回它。
### 2、作用域与执行上下文
**执行上下文**：执行上下文分为全局上下文、函数上下文和块级上下文。变量或函数的上下文决定了它们可以访问哪些数据，以及它们的行为。每段代码在执行的时候，都会先创建一个上下文，当前上下文内搜索变量值，若找不到则会往外层栈搜索，直到全局上下文，若找不到一般会报错。上下文在代码执行完毕后会做销毁处理，而全局上下文会在退出程序前销毁。每个函数在执行的时候都会先创建一个它的上下文栈，函数执行完毕后，上下文栈会弹出该函数上下文，将控制权返回给之前的执行上下文。每个函数执行完毕后都会被回收，除非它内部的变量被另外一个执行中的函数引用，则未被引用的会被销毁，这个时候就会产生`闭包`，那么问题来了，什么是`闭包`?我所理解的`闭包`就是：**当前函数有权限访问另外一个作用域范围内的变量** ，即另外一个函数在执行完毕后内部的变量应该被回收销毁掉，但是由于被当前正在执行的函数使用了部分变量，导致原函数内部部分变量暂时没有被销毁。<br />
**作用域链**:在试图访问一个变量的时候，JavaScript解释器会先在当前作用域范围内查找，若没有找到，就往父辈作用域继续查找，若父辈没有则层层往上，直到全局作用域中。每一个函数在执行的时候，都会拷贝上级作用域，形成作用域链。那么这个时候就明白`闭包`产生的本质了：`当前环境中存在指向父级作用域的引用`。
### 3、如何合理的使用内存
- 合理的使用闭包，及时将不必要的变量设为`null`；
- 避免在全局作用域内增加变量，若必须，则在使用完毕后`delete`掉；
## 六、一些问题
### 1、0.1+0.2为何不等于0.3？
```js
console.log(0.1+0.2);  // 0.30000000000000004
```
计算机内部用的是二进制，浮点数0.1和0.2在转换成二进制后会无限循环，而程序是有位数限制的，不可能让它无限循环下去，这个时候就会`截断`二进制出现精度的损失，所以截断后的二进制在相加后再转换成十进制的时候就会出现误差。
### 2、对象属性类型隐式转换
下面这段代码输出什么？
```js
var obj = {a:1};
var obj2 = {b:2};
obj[obj2] = 3;
obj[{c:3}] = 4;
console.log(obj[obj2]);
```
JavaScript对象属性是字符串格式，若不是，则会发生隐式转换。所以在执行`obj[obj2] = 3;`的时候，`obj2`会调用`toString`方法转换成`[object Object]`，则相当于`obj["[object Object]"] = 3`。所以执行第四句的时候相当于执行第三句，所以答案是 `4`
### 3、[] == ![]结果是什么？为什么？
如果你仔细看过本文，则本答案一定会知道的。等号右侧返回的是一个布尔值，则会将两侧的值隐式转换成数值再比较。右侧布尔值是`false`，转换成数值为`0`。左侧空数组转换成数值`Number([])`得到的结果是`0`，所以最终结果是`true`
### 4、JavaScript内存中数据的存储方式
一般来说，系统会划分两种不同的内存空间：栈(stack)和堆(heap)。基本数据类型会按照一定的顺序存放在`栈`中。引用数据类型一般会无序的存放在`堆`中，没有数据结构，数据可以任意存放。<br />
`栈`中的数据结构，有一种`块状`的味道在里面，所以每次在复制一个变量给另外一个变量的时候，都会拷贝一份新的出来，从而与之前的数据完全脱离关系。而`堆`中的数据无序排列，当你当你试图`复制`一份出来的时候，它似乎很`懒`，并不会真的在`堆`中完全复制一份，只是告诉需要复制的变量放也`指向`当前堆中。
### 5、下面代码返回结果是什么？
```js
let a = 1;
let b = new Number(1);
let c = 1;
console.log(a == b);
console.log(a === b);
console.log(b === c);
```
```js
function sum(a, b) {
  return a + b;
}
var result = sum(1, '2');
console.log(result);
```
答案很简单，大家可自行运行结果。