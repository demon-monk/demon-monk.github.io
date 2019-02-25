---
layout: post
title: ESM Live Binding
tag: Javascript
comment: true
---

> commonJS 和 esModule 应该是我们日常最常用到的两个 js 模块规范。这篇文章以跟 commonJS 全程对比的方式，解释了：
>
> - 啥是 live binding；
> - 为啥要 live binding；
> - live binding 怎么实现的；

## commonJS 和 ESM 在导入导出模块上的区别

话不多说，先通过一个例子看下 commonJS 规范和 esModule 的一个差别。

### commonJS import & export

解释之前先看个 commonJS 的 🌰

```js
// lib.js
var mutableValue = 3;
var mutableObj = {
  value: 3
};
function incMutableValue() {
  mutableValue++;
}

function printMutableValue() {
  console.log(mutableValue);
}

function printMutableObjValue() {
  console.log(mutableObj.value);
}

function incMutableObjVal() {
  mutableObj.value++;
}
module.exports = {
  mutableValue, // value copied 🔥（1）
  mutableObj, // addr copied 🔥（2）
  incMutableValue,
  incMutableObjVal,
  printMutableObjValue,
  printMutableValue
};
```

```js
// main.js
var {
  mutableValue, // value copied 🔥（1）
  mutableObj, // addr copied 🔥（2）
  incMutableValue,
  incMutableObjVal,
  printMutableObjValue,
  printMutableValue
} = require("./lib");

// The imported value is a (disconnected) copy of a copy
console.log(mutableValue); // 3
incMutableValue();
console.log(mutableValue); // 3 💡（1）
printMutableValue(); // 4

// The imported value can be changed
mutableValue++;
console.log(mutableValue); // 4 💡（2）
printMutableValue(); // 4

console.log("================");

console.log(mutableObj.value); // 3
incMutableObjVal();
console.log(mutableObj.value); // 4 💡（3）
printMutableObjValue(); // 4

mutableObj.value++;
console.log(mutableObj.value); // 5 💡（3）
printMutableObjValue(); // 5
```

💡（1）：调用`incMutableValue()`后，在 main.js 中`mutableValue`的值为 3，而在下一行我们可以看到在 lib.js 模块内部`mutableValue`的值已经变成了 4。

💡（2）：在 main.js 中增加`mutableValue`的值，对 lib.js 中`mutableValue`也没有影响

> 说明`mutableValue`在被导入到 main.js 模块后就已经跟导出它的模块失去了关联，在各自模块中修改这个变量的值，不会影响其他模块的值
> 🔥（1）：这是因为 commonJS 导出`mutablValue`时，在 lib.js 模块内部对这个变量进行了一次拷贝，而在导入时，又在 main.js 中对这个变量进行了一次拷贝。经过两次拷贝后，两个模块中的变量虽然同名，但其实已经没有什么关系了。

💡（3）：`mutableObj`导出后看起来在两个模块中，值是同步更新的。

> 🔥（2）:其实原理跟 🔥（1）是一样的。只不过因为导出的是一个对象，所以拷贝的不是值，而是对象的地址。所以两个模块中的值才能同步更新。但这改变不了一个事实，就是在 commonJS 中，变量一旦被导出，导出模块和引入模块中的同一个变量就失去了联系。换个高大上一点的说法就是**commonJS 模块规范不支持 live-binding**

### esModule import & export

```js
// lib.mjs
export let mutableValue = 3;
export function incMutableValue() {
  mutableValue++;
}
export function printMutableValue() {
  console.log(mutableValue);
}
```

```js
import { mutableValue, incMutableValue, printMutableValue } from "./lib.mjs";
console.log(mutableValue); // 3
incMutableValue();
console.log(mutableValue); // 4 💡（1）
printMutableValue(); // 4 💡（1）

mutableValue++; // TypeError: Assignment to constant variable. 💡（2）
console.log(mutableValue);
```

💡（1）：在 esModule 规范下，变量在导出后，在导出模块进行的更改是可以同步更新到导入模块的。

💡（2）：在 esModule 下，变量一旦被导入，就不允许被修改

> 总而言之，**esModule 是支持 live-binding 的**

## Why live-binding

> **为了解决循环引用**

### commonJS 循环引用

话不多说，先看 commonJS 下一个循环引用的例子

```js
// aUtils.js
exports.a = function() {
  return "a";
};

// bUtils.js
const appUtils = require("./appUtils");
exports.b = function() {
  return "b" + appUtils.aUtils.a(); // 💡TypeError: Cannot read property 'a' of undefined
};

// appUtils.js
const aUtils = require("./aUtils");
const bUtils = require("./bUtils");
module.exports = {
  aUtils,
  bUtils
};

// app.js
const appUtils = require("./appUtils");
console.log(appUtils.bUtils.b());
```

![](https://ws2.sinaimg.cn/large/006tKfTcgy1g0iiwxsj0hj30pw0iedhs.jpg)
💡：妥妥的循环引用，没毛病。当用`node app.js`运行时，也妥妥地报了个错`TypeError: Cannot read property 'a' of undefined`
为啥呢？

> appUtils 依赖 bUtils，所以 bUtils 得先加载，但是在加载 bUtils 的时候又依赖了 appUtils。但是前面说过，appUtils 还没有加载，上面啥都没有，所以访问`appUtils.aUtils.a`时，是找不到的。

### esModule 循环引用

同样的逻辑，用 esModule 再实现一发

```js
// aUtils.mjs
export function a() {
  return "a";
}

// bUtils.mjs
import appUtils from "./appUtils.mjs";
export function b() {
  return "b" + appUtils.aUtils.a();
}

// appUtils.mjs
import * as aUtils from "./aUtils.mjs";
import * as bUtils from "./bUtils";
export default { aUtils, bUtils };

// app.mjs
import appUtils from "./appUtils.mjs";
console.log(appUtils.bUtils.b());
```

这时候运行确成功了，控制台打印出了`ba`
为啥呢？

> 跟前面同样的依赖关系，所以 bUtils 会先求值（Evaluation），但是在求值过程中会依赖 appUtils。这时候 appUtils 还没有求值，所以上面应该啥都没有，所以在 bUtils 求值过程中 appUtils 还是个空无一物的对象。目前为止跟 CommonJS 遇到的情况一模一样。
>
> 但是 esModule 有了 live-binding，剧情在后面开始反转，等 bUtils 和 aUtils 求值完毕后，会进行 appUtils 的求值。完了之后，appUtils 开始有内容了。通过 live-binding，将其导出的值同步给所有依赖它的模块。app 模块处在依赖图的顶端，所以最后求值，所以在最终 app 模块求值的时候，所有他依赖的模块都是有内容的。😝😝😝 循环引用好像没问题了。

#### 需要注意的问题

esModule 看起来解决循环引用的问题，那我们以后用 esModule，就可以放飞自我，想怎么 import 都行，不用再绷紧循环引用这根弦了吗？
💀💀💀 不是的！
🔥🔥🔥**在求值的过程中，不能运行包含了循环引用模块的代码。**
比如上面的例子中，我们加一行代码

```js
// bUtils.mjs
import appUtils from "./appUtils.mjs";
console.log(appUtils.aUtils.a()); // 💣💥💣💥💣💥💣💥💣💥
export function b() {
  return "b" + appUtils.aUtils.a();
}
```

上面的代码依然会报错的。
为啥呢？

> 其实前面已经说得很清楚了。bUtils 中的`appUtils.aUtils`，必须得等 appUtils 模块求值结束才会有值。而在后者的求值过程在前者之后的，所以在 bUtils 求值的时候`appUtils.aUtils`仍然为空。

## BTW，live binding 好牛逼，怎么做到的

嗯，既然是 ES 标准，自然从根源上是从语言层面上解决这个问题。
在[这篇文章](https://hacks.mozilla.org/2018/03/es-modules-a-cartoon-deep-dive/)中深入讲解了 esModule 加载、解析、Link、求值的全过程。我这里简单总结一下，回答一下这个标题。

- 加载这个概念好说，既然要分模块（文件）嘛，自然要把文件搞过来。不管是通过网络（浏览器），还是读本地文件（Node）。
- 解析就是把文件中的内容，进行抽象解析成引擎可理解，切便于使用的结构（Module Record）
- Linking 是实现 live-binding 的重点。在这个过程中，将不同模块的内容整合到一起。各个模块交互的点（可以认为是接口）其实就是 export 和 import。在 Linking 过程中，会为这些点开辟内存位置。(敲黑板，划重点 👨🏻‍🏫👨🏻‍🏫👨🏻‍🏫)export 点和对应的 import 点指向同一块内存，所以有任何值的更改，二者都是同步的。
- 求值，Linking 的时候不是给 export 点都开辟了内存嘛，但是没有赋值。求值过程中会按照依赖树，以深度优先的规则遍历，执行模块中的代码，执行完后给哪些内存填值。

## 🎉🎉🎉🎉

看到这儿，你应该能理解 esModule 的 live binding 是什么东西了。也应该知道使用 esModule 解决循环引用的正确姿势了。如果还有时间，建议看下下面的参考文章，理解更深入哦。

[图解 esModule](https://hacks.mozilla.org/2018/03/es-modules-a-cartoon-deep-dive/)

[esModule 到底 export 了个啥](http://2ality.com/2015/07/es6-module-exports.html)
