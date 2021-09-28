# 我所理解的js扁平化(拍平)数组的方法有哪些
**需求**：
将给定的数组拍平，即变成一维数组，你能想到哪几种方法。

```js
var arr = [1,2,[3,[4,5,[6]],7],8];
```

```js
//第一种处理方法
var tempArr = JSON.stringify(arr).split('');
var arrFlat = [];
for(var i=0;i<tempArr.length;i++){
  if(!isNaN(Number(tempArr[i]))) arrFlat.push(Number(tempArr[i]))
}
```
```js
//第二种方法
var arrFlat = arr.flat(Infinity)
```

```js
//第三种方法
var arrFlat = arr.join(',').split(',').map(item => Number(item));
```

```js
//第四种方法
var arrFlat = [];
function flatFn(ary){
  for(var i=0,l=ary.length;i<l;i++){
    if(Array.isArray(ary[i])){
      flatFn(ary[i])
    }else{
      arrFlat.push(ary[i])
    }
  }
}
```

```js
//第五种方法
while(arr.find(item => Array.isArray(item))){
  arr = [].concat(...arr)
}
```

```js
//第六种方法
function arrFlat(arr){
  return arr.reduce(function fn(pre,cur){
    return pre.concat(Array.isArray(cur)?arrFlat(cur):cur)
  },[])
}
```
利用数组的map方法我试过了，基本上跟遍历差不多。<br />
欢迎评论指出不足以及更多方法。