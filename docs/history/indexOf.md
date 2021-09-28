# js如何查找一篇文章中出现最多的字符
## 背景
记得好几年前做一个项目开发的时候，马上要下班了，被产品经理临时“难为”(插)了一次：“我想知道一篇文章中某个字、词或者语句出现的次数最多，出现了多少次？”。有点小郁闷，但也没怼回去，不然时间浪费了，产品的需求有了，我的代码还没时间写。与其这样，不如将时间用来思考。
## 需求拆解
### 1、文章可看作一段字符串，只要查找指定字符串出现了多少次即可
关键知识点：字符串 indexOf 查找方法。语法：indexOf方法可返回某个指定的字符串在字符串中首次出现的位置，若不存在，则返回 -1。
```js
stringObject.indexOf(searchvalue,fromindex)
```
+ searchValue  需要查找的字符串内容(必需)
+ fromIndex   可选整数参数，规定字符串开始检索的位置(非必需)。若取消该参数，则从最左侧开始查找。

其实大部分同学都知道第一个参数，不知道还可以从指定位置查找
```js
/**
 * @description 查找指定字符串在字符串中出现的次数
 * @param val 需要查找的字符串
 * @param str  目标字符串查找范围
 */
function search(val,str){
    if(arguments.length < 2) return
    var index = str.indexOf(val);
    var nums = 0;        //出现的次数
    while(index !== -1){
        //从找到的位置继续往后搜索，每次都记录一条
        nums++
        index = str.indexOf(val,index+1)
    }
    return nums
}
```
### 2、一篇文章中有多少字符需要检索呢？

在代码层面看来，连续的字符都可能是词语，所以干脆将连续的词语都拆解遍历后去查找。例如字符串  "abc"有可能需要检索多种可能  "a"、"ab"、"abc"、"b"、"bc"、"c"。
```js
var str = "abc";
var len = str.length;
var strArr = [];
for(var i=0;i<len;i++){
    for(var j=1;j<=len;j++){
      var ele = str.substr(i,j);
        if(ele && ele !== strArr[strArr.length-1]) strArr.push(str.substr(i,j))
    }
}
```
### 3、完成了单个字符串的查找，又解决了需要查找多少个字符串，剩下的就好办了
```js
var maxStr = '',maxLen = 0;
for(var i=0;i<strArr.length;i++){
  var nums = search(strArr[i],str);
  if(nums > maxLen){
    maxStr = strArr[i];
    maxLen = nums;
  }
}

console.log(maxStr)
console.log(maxLen)
```
## 方法二
```js
var str = 'abcebdefgb';
var len = str.split('b').length-1;
```
## 总结
将以上三段代码复制到一起即可运行，虽然不是最好的方法，但是可以帮助入门的同学增强思维方式。