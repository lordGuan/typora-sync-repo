## 可迭代协议

这个协议约束对象定义自己的的迭代行为，`for...of`这种循环结构，就是在被循环对象上进行迭代访问，这就依赖其定制的迭代行为。类似Array和Map都有自己的迭代行为，普通的对象并没有提前定义迭代行为。可迭代对象要求实现**@@iterator**方法，即访问对象的[Symbol.iterator]属性必须是可行的，对象本身或者其原型链上必须有这个属性。

[Symbol.iterator]这个属性的值应当是一个函数，返回一个符合`迭代器协议`的对象。需要被迭代的时候，**@@iterator**方法被无参调用，返回一个“迭代器实例”，用这个实例来获取每次访问的值。

可迭代协议实际上是规定了可迭代与不可迭代对象之间的差异，由于JavaScript没有Interface（接口）的概念，必须要通过协议的方式，来约束具有特定行为的对象应当具备的特性。

## 迭代器协议

可迭代协议是要求对象在需要被迭代时，如何实现自己的迭代行为；迭代器协议则是约定了一种标准的方式，可以产生有限或无限序列的值，还要求在迭代的最终总会有一个默认的返回值。所以说迭代器协议，是定义了怎样才算是一个迭代器。可以参考下面的**迭代器**一节。

迭代器可以显示的调用next()来逐个获取值，并通过done属性来判断是否完成，这个动作就是一个迭代行为。所以一般来说可迭代协议和迭代器协议是同时实现的。用迭代器协议的实现，作为可迭代的默认行为。最简单的迭代器：

```js
var myIterator = {
    next: function() {
        // 自己定义如何返回下一个值，相当于决定访问序列的顺序和限度
    },
    [Symbol.iterator]: function() { return this } // 自身就是符合迭代器协议的对象
}
```

由于[Symbol.iterator]这个属性通常是不会显示的访问，且next()方法也有可能不会暴露出来，比如Array的实例，可以`for...of`进行迭代，但是next()并不是直接访问的，所以提供了一些方法来将使用的迭代器暴露出来。

```js
let values = [1,2,3,4,5].values(); // 这里values就是数组内涵的迭代器
// 这里还可以使用.entries()来返回一个迭代器，这个迭代器的值是[index, ele]这种格式的
let n = values.next();
while(!n.done) {
  console.log(n.value); // 依次打印序列中的每一个值，打印出最后一个元素后，n.done === true，结束遍历
  n = values.next();
}
```

另外，`for...in`也是一种循环结构，一般针对对象的属性进行遍历（只会遍历那些“可遍历”的属性，具体可以参见defineProperty），但实际上一个普通的对象并不是可迭代的。`for...in`可以用作数组的遍历，但是顺序可能无法保证。

## 一些例子

可迭代协议里要求了**@@iterator**的实现，所以可以用这一点来判断一个对象是否可迭代：

```js
let ss = "hello";
typeof ss[Symbol.iterator]; // 这里应当返回"function"

let iterator = ss[Symbol.iterator](); // 获取到字符串的迭代器实例
// console.log(iterator.toString()) 这里会返回[object String Iterator]，表示它确实是一个迭代器

iterator.next(); // {value: 'h', done: false} 
// 显示的调用next()，可以发现字符串实际上是字符的序列
// 可迭代的对象一般都可以使用展开语法
[...ss]; // 你会得到['h','e','l','l','o'] 
```

为了让用户自己实现可迭代协议，[Symbol.iterator]属性是可以修改的，因此也可以覆盖已经实现的迭代行为：

```js
let ss = new String("hello"); // 这里必须使用显示构造String实例，避免自动装箱
ss[Symbol.iterator] = function() {
  // 返回一个实现迭代器协议的对象
  return {
    next() {
      // 必须返回{value: any, done: boolean}
      return {value: "+" + ss[this._index++], done: this._index >= ss.length}
    },
    _index: 0
  }
};
console.log(iterator.next().value); // "+h"
console.log([...ss]); // [ '+h', '+e', '+l', '+l' ] 
```

*对于string，number和boolean这三种基础数据类型，js有对应的三种包装类型String，Number和Boolean，但你以字面量的形式声明了变量，如let ss = "hello"，并执行了ss.substring(2)，String.prototype.substring实际上是String封装类实例的方法，所以JavaScript会创建对应的包装类型，然后调用指定的方法，然后会销毁掉的。所以如果不显示的创建包装类型，你是无法覆盖[Symbol.iterator]的，因为都覆盖到临时的那个装箱变量上。*



## 迭代器

[原文](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Iterators_and_Generators#Iterators)

> In JavaScript an **iterator** is an object which defines a sequence and potentially a return value upon its termination

在JavaScript中，迭代器是这样一个对象：定义了一个序列，在终止时可能会返回一个值。更确切的说，迭代器是实现了[**迭代协议**](#可迭代协议)，要求是拥有一个next()方法，这个方法返回一个对象，包含`value`属性--序列中下一个值，和`done`属性--true表示序列中最后一个值已经被访问过。如果`value`和`done`都有，其被称为迭代器的返回值。

迭代器一旦被创建，可以通过重复的调用next属性来显式的迭代。迭代器的迭代过程可以被称为是“消耗迭代器”，因为通常这个过程只执行一次。（原文比较拗口，其实就是表达迭代一个集合，相当于单程的地铁，一次经过一个站）。当最终值已经产生，额外再调用next()应当只是返回{done:true}（只有`done`属性没有`value`属性，已经不能算作迭代器的返回值了）。

JavaScript中最常见的迭代器就是数组迭代器，单纯的按照顺序返回关联数组中的每一个值。很容易去想象，所有的迭代器都可以被描述成数组，但实时并非如此。数组必须按照体量完整分配空间，但是迭代器只在需要的时候才被消耗，因此可能描述成一个不限大小的序列，比如0到无穷之间所有的整数形成的集合（即特定情况下，迭代器的next()是可以无限调用下去，永远有下一个值）。

下面用一个例子来说明。这个例子可以创建一个简单的范围迭代器，根据提供的起始点（start和end，左闭右开区间）来创建一个整数步长序列。最终返回值是创建序列的大小（iterationCount）。

```
// start 包含左端点 end 不包含右端点 step 整数间隔步长
function makeRangeIterator(start = 0, end = Infinity, step = 1) {
    let nextIndex = start;
  	// 闭包内的用于计数的变量
    let iterationCount = 0;
		
  	// 具备迭代器的定义特性，有一个next方法，next返回的对象有value属性和done属性
    const rangeIterator = {
       next: function() {
           let result;
           if (nextIndex < end) {
               result = { value: nextIndex, done: false }
               nextIndex += step;
               iterationCount++;
               return result;
           }
           return { value: iterationCount, done: true }
       }
    };
    return rangeIterator;
}

// 使用方法
let it = makeRangeIterator(1, 10, 2);

let result = it.next();
while (!result.done) {
 console.log(result.value); // 1 3 5 7 9
 result = it.next();
}

console.log("Iterated over sequence of size: ", result.value); // [5 numbers returned, that took interval in between: 0 to 10]
```

*注：无法通过反射判断对象是否可迭代*

## 生成器函数

迭代器next方法的实现决定了如何访问集合中的元素，对于集合的操作需要非常谨慎，不能因为一个迭代的行为而导致集合产生了不可预期的影响。除此之外，next方法调用的时机不确定，两次next的调用期间，要维护集合的稳定。所以迭代器对于内部状态的维护非常重要。生成器函数是迭代器的一个可选替代方式，通过定义一个不连续执行的函数，定义迭代算法。

### 生成器函数的语法

```js
function* funcName(...args): Generator {}
```

### 生成器函数的行为

调用生成器函数并不会执行函数体的任何代码，而是返回一个特殊的迭代器**Generator**--生成器。生成器调用next方法想要消耗一个值时，生成器函数体才会执行，并且在遇到**yield**时停下，将**yield**的操作数作为value返回，如果将所有的yield都消耗完了，将返回{value: undefined, done: true}作为迭代的终止。

多次调用生成器函数会得到全新的生成器，相互之间不会相互影响，但是都只能迭代一遍。

之所以叫**Generator**生成器，就是因为它能在调用next()时，按照既定的规则生成一个值，而不一定需要既定集合去迭代。*yield*单词本身有“产生，产出”的含义，作为关键字其操作数会作为迭代的结果被“产生”。

来自官网的例子：

```js
function* makeRangeIterator(start = 0, end = 100, step = 1) {
    for (let i = start; i < end; i += step) {
        yield i;
    }
}
```

用工厂方法的形式也可以实现类似的功能：

```js
function rangeIteratorFactory(start = 0, end = 100, step = 1) {
  return {
    next() {
      let done = this._currentValue >= end,
        result = {
          value: done ? undefined : this._currentValue,
          done
        };

      this._currentValue += step;

      return result;
    },
    _currentValue: start
  }
}
```

但是返回的终究是个普通的对象，只是实现了迭代器协议，正确使用也可以得到预期的结果。

### 高级生成器

生成器返回一个迭代器，按函数体中yield来完成迭代。每次调用next都会回到函数体上次结束的位置继续执行，直到下个yield语句。对于外部来说，调用next来获取，或者说消费下一个值，特定次数的调用，理论上结果应该是一样的。比如一个斐波那契数列生成器，使用一个无限循环来求数列的下一项：

```js
function* fibonacci() {
  var fn1 = 0;
  var fn2 = 1;
  while (true) {  
    var current = fn1;
    fn1 = fn2;
    fn2 = current + fn1;
    yield current;
  }
}
```

只要通过这个生成器获得的迭代器，第三次调用next()都会返回value为1的结果。而生成器的工作流程可以发现有一个回到上次执行结尾的特性，相当于我们调用next()时，会从生成器函数上一次yield的位置之后继续执行。所以有的时候用一个“执行权”的概念来描述生成器函数的执行。借官网的一段代码：

```js
let sequence = fibonacci();
console.log(sequence.next().value);     // 0
console.log(sequence.next().value);     // 1
console.log(sequence.next().value);     // 1
```

首先获取了生成器对应的迭代器，第一次调用next()，“执行权”从生成器函数的首行开始，初始化fn1和fn2，进入循环，创建变量current，赋值为fn1的值，操作fn1和fn2（分别为1和1），遇到yield将current的值组装到迭代器返回的结果中，将“执行权”交回到第一次调用next()的调用处，访问value属性并打印。“执行权”交出去了，但是执行环境还是保留着的，所以到第二次调用next()时，“执行权”回到生成器函数中上次停下的地方，即fn1和fn2分别为1和1的时候继续执行，回到循环头，再声明一个current并赋值，操作fn1和fn2（分别为1和2），遇到yield将current返回，再次交出“执行权”，外部得到结果1。后面的就依次类推。

不难发现“执行权”交回的时候，外部并无法影响其内部逻辑，对于斐波那契数列来说，将无限迭代下去。除非我们重新获取一个迭代器，否则我们无法重新开始。所以迭代器的next()函数，可以传入一个参数，用于执行权交回的时候，给定一个信号。这个信号在执行权交回的时候，作为yield语句的值，可能参与到后续的逻辑，所以我们再从官网拿个例子：

```js
function* fibonacci() {
  var fn1 = 0;
  var fn2 = 1;
  while (true) {  
    var current = fn1;
    fn1 = fn2;
    fn2 = current + fn1;
    var reset = yield current;
    if (reset) {
        fn1 = 0;
        fn2 = 1;
    }
  }
}
```

当我们调用next(true)时，变量reset将被赋值为true，后续就可以重启序列了。生成器函数中yield将其操作数求值作为迭代的value返回，而next()传入的参数则作为执行权交回生成器函数式，整个yield语句的值，参与到后续的逻辑中。

除此之外，生成器函数返回的迭代器，通过调用throw()来制造一个异常，在生成器函数内部被捕捉，产生异常和异常的处理并不在一起。

### yield*

```js
yield* [[expression]];
```

这个表达式可以认为是迭代器的嵌套，用另一个迭代器的迭代情况来补充自身的迭代。拿一个官网的例子：

```js
function* g1() {
  yield 2;
  yield 3;
  yield 4;
}

function* g2() {
  yield 1;
  yield* g1();
  yield 5;
}
```

在遇到yield*时，将从其操作数中进行迭代。也可以认为是将执行权进行一次传递。可以将“产生”这一动作委托给其他的可迭代对象，你可以yield\* [1,2]，则从数组中依次产生值。

## 生成器函数与异步

前面有提到生成器函数的行为，可以用执行权的交换来理解，这已经有异步的意味在里面。**异步**与同步相对，不阻塞线程来完成任务，宏观上看，一个任务的执行，往往不是在一段连续的时间内执行并完成的。而生成器函数的函数体，在调用next()时，可以从上次停下的地方继续执行，也就是说，整个函数体就是一个异步任务，在特定的时候停下交出执行权，又在特定的时候得到执行权继续完成。

JavaScript引擎是在单线程上执行的，为了实现异步编程，通常采用回调函数、事件监听、Pub/Sub和Promise等方式实现。**协程**的概念在很多语言中，也是异步编程的方案之一。协程也被当做轻量的线程，简单来看就像是函数间的切换执行。Generator函数就类似协程，通过yield来交出执行权，暂停自身的执行，等到特定的时候继续，所以说生成器函数体就是一个异步任务。生成器函数返回的迭代器，就相当于这个异步任务执行的控制器，调用next()来让生成器函数逐步执行，并且返回阶段性的信息，value分段执行的结果，done任务是否完成。

生成器函数可以停止可以恢复，是封装异步任务的基础，除此之外还为他设计了数据交换的方式（next().value和向next()传入参数），异常的产生与捕捉，都为生成器函数模式的异步编程提供支持。

### 用生成器函数封装异步任务

生成器函数利用执行权交换的特性，可以改变异步任务的宏观流程，偷了一个例子过来：

```js
function* gen () {
  var url = 'https://api.github.com/users/github'
  var result = yield fetch(url)
  console.log(result.bio)
}

var g = gen()
var result = g.next()

result
  .value
  .then(function (data) {
    return data.json()
  })
  .then(function (data) {
    g.next(data)
  })
```

可以看到gen函数体，不看yield基本上就是一个同步的处理流程，g.next()启动，拿到fetch返回的promise实例，在then方法中对数据进行json化，再通过next(data)将执行权交回，所以让gen的函数体看上去符合一般的逻辑流程。但是这样的写法会有一定的混淆，因为真正异步任务完成和结果被处理并不在一起。

### Thunk函数

Thunk在专业释义中表达为“形（式）实（在）转换程序”。JavaScript中Thunk函数是高阶函数的一种，通常是用来将已有的函数，结合新的参数，形成新的函数。总的来说Thunk的作用是“替换”，偷一个例子来看：

```js
fs.readFile(fileName, callback);

var Thunk = function (fileName) {
  return function (callback) {
    return fs.readFile(fileName, callback);
  };
};

var readFileThunk = Thunk(fileName);
readFileThunk(callback);
```

readFile本来是一个多参数函数，Thunk将fileName作为替换内容，生成一个只接受callback的函数。所以你在调用readFileThunk时，其实都是提前将fileName参数替换掉的过程，或者说将readFile的调用替换掉了。具体可以参见Thunk的详细说明，简单理解成“传名调用”也可以，是一种替换表达式的实现方式。

其实Thunk函数的最内层return一定是要被替换的最原始多参数函数，然后用嵌套return function()来创造上下文接受部分参数，作为上层的替换结果。由于每一层return function()将多参数打散，是的在调用的时候会有Thunk(fileName)(callback)这样的情况，直到最后一个调用才真正执行，而前面的调用，相当于将参数进行缓存，有一个延迟求值的作用。

### 生成器函数的执行控制

从生成器函数的行为特性可以看到，整个执行流程实际上非常依赖外部对于next的调用，如果外部不知道生成器内部需要多次完成，可能还需要外部通过永真循环来让生成器函数彻底执行完。并且生成器函数包含多个异步操作时，后一阶段要依赖上一阶段，外部可能就需要进行then的嵌套。所以这个嵌套可以用递归的方式使其自动完成，而不需要知道实际有几个阶段。偷了一个例子：

```js
function run(fn) {
  var gen = fn();

  function next(err, data) {
    var result = gen.next(data);
    if (result.done) return;
    result.value(next);
  }

  next();
}

function* g() {
  // ...
}

run(g);
```

run()方法接受一个生成器函数，函数体内先得到生成器对应的迭代器gen，定义递归函数next()，这个函数的作用就是控制迭代器消耗。不难发现这个next()其实就是一个指令，告诉生成器函数生成异步操作完成后继续向后就是了。控制生成器函数的执行，就是控制好执行权的转移，上面的例子是通过回调函数递归实现自动的执行权转移，换成Promise也可以通过then进行递归实现。

生成器函数实际上就是异步操作的容器，遇到异步操作将执行权交出，完成后应当重新得到执行权进行下一个异步此操作。所以生成器函数的自动执行的关键，就是在异步操作完成时能够主动的交还执行权。上面的例子中，就是通过next()方法的递归来实现自动的交还。总结起来，1. 回调函数型异步操作，包装成Thunk函数，回调中交还执行权。2. Promise型异步操作，then方法注册的回调进行交还。

## async/await 函数

### AsyncFunction

async function(){} 就是一个AsyncFunction的实例。Object.getPrototypeOf(async function(){}).constructor可以获取构造方法（虽然这样做并没有意义）。

```js
let AsyncFunction = Object.getPrototypeOf(async function () {}).constructor;

// 使用语法，用不用new都可以，且不创建闭包，上下文始终是全局
AsyncFunction(...args:any[], functionBody:string): function;
```

这个并不重要，只是提一下。

`异步函数`是通过事件循环异步执行的函数（因为一般的函数都作为单独的调用栈同步执行），隐式返回Promise代表其执行结果。异步函数语法和结构上更像是同步执行流程，比如官网的例子：

```js
function resolveAfter2Seconds() {
  return new Promise(resolve => {
    setTimeout(() => {
      resolve('resolved');
    }, 2000);
  });
}

async function asyncCall() {
  console.log('calling');
  var result = await resolveAfter2Seconds();
  console.log(result);
  // expected output: 'resolved'
}

asyncCall();
```

asyncCall的逻辑理解顺序，是等到resolveAfter2Seconds()函数对应的Promise解决了之后得到结果result在继续执行。已经看到了生成器函数的影子，await实际将执行权交出去了，promise解决了以后将执行权交回并继续。所以从流程上看更像是同步的。

### 语法

```js
async function name(...args) { statements }
async (...args) => { statements }
async function(...args) { statements }

let ob = {
  async func(...args) { statements }
}
```

可用于匿名函数和箭头函数。

隐式返回一个Promise实例，函数体执行完成则对应resolve，函数体异常则对应reject。

### 描述

异步函数才可以包含`await`指令，该指令实际上会暂停异步函数的执行，交出执行权，待await指定的操作完成（可以参见Promise的解决）继续执行。所以只关注异步函数的流程，看上去变成了同步的过程，实际上通过执行权的转移，仍是异步的完成。实际上async/await只是展开了Promise.then嵌套回调的格式，不改变具体的执行顺序。

`await`原意就是“等待，期待”，在异步执行函数中，表达“当等待的东西得到结果了，我在继续执行”。

异步执行函数的return会隐式的传递给Promise.resolve，所以return await promiseValue和return promiseValue都是可以的，但是逻辑上是完全不同的。

### 和生成器函数的关系

一说async/await是生成器函数的语法糖。偷一下阮一峰的例子：

```js
const readFile = function (fileName) {
  return new Promise(function (resolve, reject) {
    fs.readFile(fileName, function(error, data) {
      if (error) return reject(error);
      resolve(data);
    });
  });
};

const gen = function* () {
  const f1 = yield readFile('/etc/fstab');
  const f2 = yield readFile('/etc/shells');
  console.log(f1.toString());
  console.log(f2.toString());
};

const asyncReadFile = async function () {
  const f1 = await readFile('/etc/fstab');
  const f2 = await readFile('/etc/shells');
  console.log(f1.toString());
  console.log(f2.toString());
};
```

function*和async function对应，await和yield对应。不过生成器函数返回的是迭代器，且自动执行需要显示操作，而异步执行函数隐式返回promise是自动完成的。异步执行函数像普通函数一样直接调用，就会自动执行，不需要生成器函数那样的自己实现。且async/await有更好的理解，流程上更加清晰。同时也可以将async function理解成将函数体包装成promise，而await则是将then绑定的回调进行展开。如果把await之后的语句当做当前promise.then绑定的回调，就可以发现其实多个异步操作包含在异步执行函数中，是嵌套的promise。

### 异常处理

普通promise异步操作用then处理结果，用catch处理异常，多个异步操作也可以用Promise.all进行包装处理。如果异步执行函数中使用await移交了执行权，解决异常就应当类似生成器函数中处理异常，使用`try...catch`结构来捕捉。

### 注意

虽然async/await并没有改变异步执行的特性，但是执行顺序是被确定的，所以连续的await如果不存在依赖关系（使用Promise形式不存在嵌套时），最好不要都使用await，因为异步操作的启动相当于是阻塞的。后面的操作启动的要更晚，这明显不利于异步处理效率。这种情况用await Promise.all()更有效率。

再循环中使用await也要注意，在借用阮一峰的几个例子：

```js
function dbFuc(db) {
  let docs = [{}, {}, {}];

  docs.forEach(async function (doc) {
    await db.post(doc);
  });
}

async function dbFuc(db) {
  let docs = [{}, {}, {}];

  for (let doc of docs) {
    await db.post(doc);
  }
}
```

第一个函数可能得到错误的结果，因为forEach是并发的，针对docs的每个元素执行异步函数。但第二个函数中，显示的等待post的结果，会阻塞for循环，从而确保结果的正确。

另外，由于async/await具有生成器函数的执行特性，遇到await会将执行权交出去，并保留自己的执行上下文，使得特定情况下函数的上下文得以保留，比如阮一峰的例子：

```js
const a1 = () => {
  b().then(() => c());
};

const a2 = async () => {
  await b();
  c();
};
```

调用a1以后，b()执行一个异步操作，绑定回调()=>c()，此时就不再等待，继续完成a1的执行。绑定的回调也是等事件循环来调度。而a2调用后，将执行权交到b()的异步操作，a2也并没有执行完，上下文也保留着。当然a1中绑定的回调由于创建了闭包，也能保留a1中必要的变量等。

### async的生成器函数实现

不难看出异步执行函数就是自动执行的生成器函数，所以可以用一下方式来实现：

```js
async function fn(args) {
  // ...
}

// 等同于

function fn(args) {
  return spawn(function* () {
    // ...
  });
}

function spawn(genF) {
  // 隐式返回一个promise
  return new Promise(function(resolve, reject) {
    const gen = genF();
    // 定义递归函数自动执行
    function step(nextF) {
      let next;
      // 异常处理
      try {
        next = nextF();
      } catch(e) {
        return reject(e);
      }
      if(next.done) {
        return resolve(next.value);
      }
      Promise.resolve(next.value).then(function(v) {
        step(function() { return gen.next(v); });
      }, function(e) {
        step(function() { return gen.throw(e); });
      });
    }
    step(function() { return gen.next(undefined); });
  });
}
```



实现了可迭代协议的对象就是可迭代对象，在`for...of`这种结构中，使用其迭代行为来访问集合中的值。比如Array和Map的实例都是拥有内置迭代行为的对象。Array的实例，调用\[Symbol.iterator\]()和values()都会得到遍历元素的迭代器，entries()可以参见文档。根据可迭代协议，**@@iterator**必须实现，对象本身或者其原型链上必须带有符合协议的[Symbol.iterator]属性。生成器函数返回迭代器实例，所以也可以用在[Symbol.iterator]属性上。