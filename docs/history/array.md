# 重学JavaScript系列之——数组
数组是ECMAScript中最重要的数据存储类型之一，数组是一组有序的数据，每个槽位可以存储任意类型的数据。也就是说可以创建这样一个数组：第一个元素是字符串，第二个元素是数值，第三元素是函数，第四个元素是数组，甚至存放下文即将讲到的`Map、WeakMap`数据结构。数组具有`length`值，每增加减一个元素，`length`的值也会随之变化。<br /><br />
数组内部每一项元素，其实也是按照`key:value`的结构存储的。你认为的`['a','b','c']`其实是这样存储的`[0:'a',1:'b',2:'c']`。比如你给数组新增元素时，下标若是非数值类型，则会尽量被转换为数值，若无法转换为数值，则会调用下标参数的`toString`方法转换为字符串。
```js
var arr = [];
arr[0] = 'a';   // ['a']
arr['1'] = 'b';  // ['a','b']
arr['str'] = 'valueStr'  // ['a','b','str':'valueStr']
arr[{a:1}] = 'valueObject';   // ['a','b','str':'valueStr','[object Object]':'valueObject']
arr[['1']] = 'valueArr';  // ["a", "valueArr", str: "valueStr", [object Object]: "valueObject"]
```
最后一个或许你会有点迷糊，其实是这样的：下标值`['1']`是数组，这个时候会隐式的调用数组的`toString()`，也就是`Array`原型链上的`toString`方法转换为`"1"`，然后再将字符串类型的`1`转换为数值类型的`1`，所以最后一句相当于执行了`arr[1] = 'valueArr'`。注意：每一个元素的`key`值若不是数值类型，则不会计入数组的`length`属性，所以上述数组最终的`length`值是<b>`2`</b><br />
## 一、创建数组
以下方法可以创建或者生成数组
```js
var arr = new Array(2);
var arr2 = new Array('a','b','c');
var arr3 = ['a','b','c'];
var arr4 = Array.prototype.slice.call(arguments);
var arr5 = Array.from(arguments || nodeList);   //此方法用于替换第四行ES5提供的转换arguments为数组的方法
var arr6 = Array.of('a','b','c');
```
`Array.from()`方法的第一个参数是一个类数组对象，即任何可迭代的结构，或者拥有`length`属性和可索引元素的结构，常见的有`arguments、nodeList`等。第二个为可选的映射函数，其参数结构类似数组的`map`方法，加上第二个参数等于转换为数组后再将元素一一通过函数过滤一遍。还可以增加第三个可选参数，第三个参数用于映射函数中`this`的值。
```js
function fn(a,b,c){
  return Array.from(arguments,function(item){
    return item + 1
  })
}
fn(1,2,3);  
// [2,3,4]
```
`Array.of()`方法将一组给定的数值转换成数组。
## 二、增加数组
`push()`在数组尾部增加新元素，并返回新数组的长度`length`
```js
var arr = ['a','b'];
arr.push('c','d');  // 4
```
`unshift()`在数组头部增加新元素，并返回新数组的长度`length`
```js
var arr2 = ['a'];
arr2.unshift('b','c');  // 4
```
## 三、删除数组
`pop()`删除数组最后一项，并返回删除的元素
```js
var arr = ['a','b'];
arr.pop();  // 'b'
```
`shift()`删除数组第一项，并返回删除的元素
```js
var arr2 = ['a','b'];
arr2.shift();  // 'a'
```
`splice()`支持三个参数，都是非必需的。
- 第一个参数是起始位置，第二个是删除截止位置，第三个参数用来填充删除后的坑位；
- 返回删除的元素组成的数组，若没删除则返回空数组；
- 第一个参数、第二个参数具有`顾前不顾后`的原则，也就是包含第一个位置元素，不包含第二个参数位置的元素；
```js
var arr = ['a','b','c','d'];
arr.splice(0,1);  // ['a']
arr.splice(-1);   //  ['d']
arr.splice(0,1,1);
// [1,'c']
```
若只有第一个参数，则默认删除到数组最后一位。若第一个参数为负数，则默认倒数第N位置删除，直到数组最后一位。

## 四、改变数组
以下几个方法在执行完毕后都会修改原数组(包括上面的`push、unshift、pop、shift、splice`)<br />
`reverse()`将指定的数组翻转顺序。<br /><br />
`copyWithin(beReplaceIndex,startIndex,endIndex)`在当前数组内部，将指定范围内的一组元素复制并覆盖另外一组元素，最后返回修改过的数组。最多支持三个参数，从`startIndex`位置开始，`endIndex`位置结束，复制一段数组后，再从`beReplaceIndex`位置开始替换内容。<br /><br />
`fill(newTarget,start,end)`使用给定值替换数组内部元素。也是最多支持三个参数(均为可选)，第一个参数为新替换的元素，第二个元素为指定开始替换的位置，第三个参数为结束位置。若无任何参数，则数组元素都替换为`undefined`，若只有一个参数，则默认全部替换。若只有两个参数，则默认替换到数组尾部。<br /><br />
`flat()`可以将多维数组(数组嵌套数组)“拍扁”变成一维数组。支持添加整数参数，即需要“拍扁”几维数组，就填数字几。若不知维度，可填写无穷大数组`Infinity`<br /><br />
`sort()`对数组元素进行排序，并返回排序后的数组。默认排序顺序是在将元素转换为字符串，然后比较它们的UTF-16代码单元值序列时构建的。无法保证排序的时间和空间复杂性。支持`compareFunction`参数，若为空，则按照字符串的首字母Unicode排序。若调用方法，则按照调用函数的返回值排序。<br />
- 如果 compareFunction(a,b) 小于 0 ，那么 a 会被排列到 b 之前；
- 如果 compareFunction(a,b) 等于 0 ，则 a b的相对位置不变；
- 如果 compareFunction(a,b) 大于 0 ，则 b 会排在 a 前面
## 五、查询数组
`indexOf()`查找给定元素的第一个索引值，若不存在则返回-1。语法：第一个参数是要查找的元素，第二个参数是开始查找的位置(可选)。<br /><br />
`lastIndexOf()`与上面方法类似。区别是：从尾部开始倒叙查询，返回数组中最后一个匹配元素的索引值。<br /><br />
`find()`查找并返回第一个满足函数的元素，若没有则返回`undefined`。支持第二个可选参数，为执行回调时`this`的指向。<br /><br />
`findIndex()`与上面方法类似。区别在于返回索引值，若没有则返回-1。<br /><br />
`every()`第一个参数为函数，第二个为可选的`thisArg`值(执行`callback`时使用的`this`指向)。该方法返回一个布尔值，只有每个元素都满足函数并返回`true`，最终才返回`true`，若有一项返回`false`，则终止执行后续操作，直接返回`false`。<br /><br />
`some()`与上述方法类似。区别在于只要有一项满足内部函数，则整体返回`true`。<br /><br />
`includes()`用来判断一个数组是否包含给定元素，若存在则返回`true`，否则返回`false`。<br />
## 六、迭代遍历
`forEach()`对数组的每个元素执行给定的方法，无返回值。第二个参数老样子，基本上每个支持`遍历`的方法均支持`thisArg`作为函数中this的指向(后续方法介绍略过)。注：稀疏数组会被跳过。提到`forEach`就要好奇如何跳出循环？目前来说只有一个办法跳出或终止循环，那就是抛出异常。<br />
```js
var arr = [1,2,3,4,5,6,7];
try{
  arr.forEach(function(item){
    console.log(item)
    if(item > 3){
      throw new Error('error')
    }
  })
}catch(e){
  console.log(e)
}
```
`map()`将数组中的每个元素调用给定函数后再返回一个新数组。<br /><br />
`filter()`对数组的每个元素执行给定给定函数，返回一个符合条件的新数组。<br />
## 七、转换或复制
`slice()`语法：(beginIndex,endIndex)，还是一句话可以形容，顾前不顾后。从`beginIndex`位置开始(包含)到`endIndex`位置(不含)拷贝元素出来，组成并返回一个新的数组。注意：是浅拷贝，所有原生数组提供的方法涉及到拷贝的，都是浅拷贝。该方法不会改变原始数组。<br /><br />
`concat()`此方法用于合并两个或多个数组，返回一个新数组，不会改变原数组。<br /><br />
`join()`用指定的字符串将数组的每一项连接起来，返回一个字符串。若为空则默认为`,`<br /><br />
`toString()`返回一个字符串，表示指定的数组及其元素。<br /><br />
`toLocalString()`返回一个字符串表示数组中的元素。数组中的元素将使用各自的 toLocaleString 方法转成字符串，这些字符串将使用一个特定语言环境的字符串（例如一个逗号 ","）隔开。<br />
## 八、其他高级用法
`flatMap()`也是“拍扁”数组，区别是：优先对数组执行一个`map`的功能，然后再对返回数组成员执行`flat()`，然后再返回一个新数组，该方法不会修改原数组，类似执行`arr.map().flat()`<br /><br />
`reduce()`从左向右对数组中的每个元素执行一个reducer函数，常用于累加等操作。语法：第一个参数为函数，第二个参数为初始值。注：若初始值为空，则会将数组的第一个元素作为初始值，从第二个元素开始执行函数。
```js
[1, 1, 2, 3, 4].reduce((prev, curr) => prev + curr );
```
`reduceRight()`同上述方法，区别在于从右向左执行。<br /><br />
`keys()`,`values()`,`entries()`用于遍历数组。它们都返回一个遍历器对象（详见《Iterator》一章），可以用for...of循环进行遍历，唯一的区别是keys()是对键名的遍历、values()是对键值的遍历，entries()是对键值对的遍历。`—— 摘自阮一峰的《ECMAScript 6 入门》`<br />

