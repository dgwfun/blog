js是一门非常灵活的语言，可以随心所欲的书写各种风格的代码。但是也正是因为它太灵活了，反而容易让人踩些坑， 最近整理了一些js容易犯错的问题和一些性能陷阱，在这里跟大家分享下。
1. ##### 传对象只是传引用
- ##### js中基本类型都是传值：

```js
let a = 1;
let b = a;
b = 2;
console.log(a); // 1
```
- ##### 对象的传递是传引用：

```js
let a = {val:1};
let b = a;
b.val = 2;
console.log(a.val); // 2
```

在实际的应用中，很容易忽略而修改对象属性，导致后续的逻辑出错。例如：

```js
let a = {val:{show:true}};
function funcA(a){
let c = a.val;
// ....
// 一旦这里函数逻辑过长或者c传入到其他函数中去，
// 很容易忘记c是a的引用数据而操作c对象，导致a属性意外改动。
c.show = false;
}
funcA(a);
console.log(a.val.show); // false
```
2. ##### 复杂的类型转换 
类型转换坑多主要一个原因是在于它的复杂性，下表是js基本类型间的转换对照表：

值 | 转换为：字符串 | 数字 | 布尔值 | 对象
:----------- | :-----------: | -----------: | -----------: | -----------:
undefined | “undefined” | NaN | false | throws TypeError
undefined | “undefined”	| NaN | false | throws TypeError
null | “null” | 0 | false | throws TypeError
true | “true” | 1 | | new Boolean(true)
false | “false” | 0 | | new Boolean(false)
“” |  | 0 | false | new String(“”)
“1.2” | | 1.2 | true | new String(“1.2”)
“one” | | NaN | true | new String(“one”)
0 | “0”	| | false | new Number(0)
-0 | “0” | | false | new Number(-0)
NaN | “NaN” | | false | new Number(NaN)
Infinity | “Infinity” | | true | new Number(Infinity)
-Infinity | “-Infinity” | | true | new Number(-Infinity)
1（无穷大，非零）| “1”	| | true | new Number(1)
｛｝（任意对象）| | | true	 
[]（任意数组）| “”	| 0 | true 
[9]（1个数字对象）| “9” | 9	| true	 
[‘a’]（其他数组）| 使用join（）方法	| NaN | true	 
function（）｛｝（任意函数）| |	NaN	| true	 

但是，如果你觉得有了这张表你就完全掌握了的话那就too young too simple了，很多时候会不经意间就踩坑：

```js
String(false); // "false"
Boolean("false"); // true
```
这种情况在其他语言下一般不会犯错，但是在js下就有可能犯，因为js类型转换给你的印象就是很灵活很强大，让你觉得它是支持这种的转换，但实际上它不是。

```js
Boolean(new Boolean('')); // true
Boolean(new Boolean(0)); // true
Boolean(new Boolean(false)); // true
```
因为任意对象都为true
parseInt(0.1); // 0
parseInt(0.0000001); // 1
因为parseInt的参数是字符串，如果传入的是数字，会先隐式转为字符串再转成数字，而0.0000001转换为字符串是1e-7，parseInt('1e-7')是1。

```js
parseInt('123a'); // 123
Number('123a'); // NaN
+'123a'; // NaN
```
这个是parseInt和其他转换方法的区别。
类型转换中这种显示的转换还只是小坑，真正的大坑在它的隐式转换：
```js
"true"==true; // false
2==true;//false
"2"==true; // false
"1"==true; // true
```
2\==true，你的第一印象肯定是这没毛病，绝对是true，但是你把它放浏览器里一执行，你就会感觉完全颠覆三观了，if(2)是true,而if(2\==true)却是false。
原因就在于它的隐式转换，js的规则是把true转化为Number再进行比较，true转化为数字是1，2跟1比较肯定是false了。
用\=\==能够避免很多这种隐式转换的坑，\==与\=\==的区别在于你在比较中是否允许隐式转换，我们在coding过程中应该遵循一个原则：除非你是有意的隐式转换，否则一般情况都用===。
还有一个原则是，最好不要用数组和对象来做比较，因为结果可能不可控：
```js
[] == false; // true
[] == ![]; // true
[] == ""; // true
{} == ""; // error
"" == {}; // false
2 == [2]; // true
({a:1})<({a:2}); // false
({a:1})>({a:2}); // false
({a:1})==({a:2}); // false
({a:1})>=({a:2}); // true
({a:1})<=({a:2}); // true
```
里面的坑太多，最好还是不要去趟。
3. ##### 作用域及闭包
作用域一个比较坑的地方就是作用域提升了：

```js
var foo = 1;
function bar() {
    if (!foo) {
        var foo = 10;
    }
    console.log(foo); // 10
}
bar();
```
因为这里有个作用域提升，在代码运行前，函数声明和变量定义通常会被解释器移动到其所在作用域的最顶部。
闭包中比较常见的一个例子：

```js
for (var i=1; i<=5; i++) {
    setTimeout(()=>{
        console.log(i); // 6 6 6 6 6
    });
}
```
闭包其实只要理解他的概念以后还是不容易踩坑的。

4. ##### 循环遍历
js的循环遍历方法主要有以下几种方法：for(;;)、for(in)、[].forEach()、for(of)，其中只有for(in)能遍历对象，但都能遍历数组。
那么这么多种方法可以用，我们用的时候该用哪个好呢？
下面是用Benchmark在chrome中的测试结果

因此：for(;;)>forEach=for(of)>for(in)
其中，for(in)在不需要取value值的时候效率跟for(of)是差不多的。
for(of)是es6新加的方法，用的是迭代的方法，除了可以遍历数组以外原则上是一切可迭代的对象都可以遍历。看起来一切都是那么美好，网上的很多文章也推荐用for(of)，但是js就是这么坑，看个例子：
```js
let arr = [];
arr[1] = 1;
for(let val of arr){
    console.log(val); // undefined 1
}
```
这里arr虽然是个非连续型数组，但是在for(of)中，它会根据数组的长度来遍历整个数组。
如果你还没用看到这个巨坑的话再看个例子：
let arr = [];
arr['20170724'] = 1;
for(let val of arr){
    console.log(val); // 这里将卡死你的浏览器
}
最坑的地方在于即使数组中你的key是String的，但是只要有一个key是可转化为Number的，那么它的length就有值，length有值for(of)就踩坑了。
```js
let arr = [];
arr['abcd'] = 1,
arr['efg'] = 2,
console.log(arr.length); // 0
arr['20170724'] = 1;
console.log(arr.length); // 20170725
```
所以，对于非连续型数组，千万不要用for(of)，如果你的数组是当map用，也不要用for(of)，因为一旦你的key中有个是可以转Number的，你就要被坑了。
