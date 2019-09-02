> An open standard for sound, interoperable JavaScript promises—by implementers, for implementers.

先给定个性：Promises/A+是一个开放的标准。按此标准实现可靠的、可互操作的JavaScript promise。所以Promises/A+相当于一个约定的规则，只要按照这个规则实现的promise实际上是通用的。（毕竟与JS相关的差异点已经数不胜数了

---

### 首先

> **promise**表示异步操作的最终结果。**then**方法是与Promise交互的主要方式，注册回调来接受Promise的最终值或失败的原因。

给Promise一个大体上的表述：代表异步操作的最终结果（未来的值），有一个then方法，then方法要接受回调，回调的签名大概是**data => void**或**reason => void**。既然promise是一个异步操作的最终结果，那么同步过程中应该是拿不到promise的状态，也改变不了Promise的进程。

其他内容大概就是说：then是promise可操作的基础，所以实现规范的Promise必须提供then，A+规范是A规范的更进一步的版本，规范不会牵扯创建、完成和拒绝promise，主要关注then。

### 技术术语

- promise：是一个带有行为符合本规范的**then**方法的object或function。
- thenable：是一个object或function，定义了一个**then**方法。
- value：（值）是任何JavaScript的合法值，包含undefined，thenable或promise。
- exception：是一个通过throw语句抛出的值。
- reason：是一个代表promise被拒绝原因的值。

主要是value（值）可以是所有JS合法值，包括undefined，对应resolve()。reason的定义也是一个值，所以reject也可以传递任何值。而thenable是一个很奇怪的定义，拥有then方法的对象或方法，主要是因为这样的对象可能可以被转换为promise。

### 要求

#### 状态的要求

> promise必须处于pending（进行中），fulfilled（已完成）和rejected（已拒绝）三种状态之一。

1. 当promise处于pending状态时：
   - 可以转换成fulfilled**或**rejected状态。

2. 当promise处于fulfilled状态时：

   - 不能再转换状态。

   - 必须有一个不可变的值，即promise最终代表的值。

3. 当promise处于rejected状态时：

   - 不能再转换状态。

   - 必须有一个不可变的值，代表被拒绝的原因。

不可变的值由“===”确定，所以不要求深层次不可变。白话文就是最终值如果是值类型就值不变，引用类型就引用不变。但是不要求引用类型内部字段的不变。这个要求主要是因为异步操作的结果是多少就是多少，比如网络请求得到什么值就是什么值，读取文件读到什么就是什么。也就是说如果promise的task如果包含两个异步操作，结果返回时会调用resolve，如果没有这个要求，那么promise得到一个返回时进入fulfilled状态，另一个再返回将promise的最终值修改了，这对于then注册的函数来说是不可思议的。**所以resolve是唯一且屏蔽的（reject也类似）**。

#### then方法的要求

> promise必须提供then方法访问其当前/最终的值/原因。promise的then方法要接受两个参数：
>
> ```js
> promise.then(onFulfilled, onRejected)
> ```

1. onFulfilled和onRejected都是可选的，代表着你不想处理promise的最终结果或被拒绝的原因。如果不是function将会被忽略，代表着你要处理就好好处理。

2. 如果onFulfilled是一个function：

   - 必须在promise完成后调用，传入promise的值作为第一参数。

   - 不能在promise完成之前调用。

   - **只能调用一次**

3. 如果onRejected是一个function：

   - 必须要promise拒绝后调用，传入被拒绝的原因作为第一参数。

   - 不能再promise被拒绝之前调用。

   - **只能调用一次**

4. 在执行上下文堆栈仅包含平台代码之前，不得调用onFulfilled或onRejected【1】。**先简单理解为：onFulfilled和onRejected会被加入MicroTask队列**

5. onFulfilled和onRejected作为函数执行【2】，一般this指向global，严格模式为undefined。

6. **then**方法可以在同一个promise上重复调用：

   - 如果/当promise完成时，所有相应的onFulfilled必须按照多个then注册的顺序执行。
   - 如果/当promise拒绝时，所有相应的onRejected必须按照多个then注册的顺序执行。

7. **then**方法必须返回一个promise【3】：

   ```js
   promise2 = promise1.then(onFulfilled, onRejected);
   ```

   - 如果onFulfilled或onRejected返回一个值x，要执行Promise处理流程**\[[Resolve\]\](promise2, x)**。
   - 如果onFulfilled或onRejected抛出一个异常e，promise2必须拒绝，且原因为e。
   - 如果onFulfilled不是一个function，且promise1完成了，promise2也将完成，且和promise1拥有相同的值。
   - 如果onRejected不是一个function，且promise1拒绝了，promise2也将拒绝，且和promise1拥有相同的原因。

关键点一个是onFulfilled和onRejected的调用顺序及调用次数，一个是then要返回promise。promise的状态转移的确定性，要求onFulfilled或onRejected回调在特定的时机，只执行一次。多次执行代表着状态转移的不确定性。

#### Promise Resolution Procedure

我称其为**Promise解析流程**。这是一个抽象的操作，将promise和value作为输入，表达为**\[[Resolve]](promise, x)**。如果x是一个thenable，假设x的行为与promise有一定程度上的相似，它会尝试去使用x的状态创建一个promise。否则的话它将用x完成promise。

只要thenable暴露一个符合Promise/A+规范的then方法，就可以将thenable在一定程度上当做promise对待，让promise的具体实现具备互操作性。

**\[[Resolve]](promise, x)**执行以下步骤（用伪代码说明）：

```
let then;
if(promise === x) {
	reject promise with TypeError;
}

if(x is Promise) {
	// 如果x是一个promise，接受x的状态[4]
	switch(x.state) {
		case 'pending':
			promise.state = 'pending'; // 直到x的状态发生变化
			break;
    case 'fulfilled':
    	promise.state = 'fulfilled';
    	promise.value = x.value;
    	break;
    case 'rejected':
    	promise.state = 'rejected';
    	promise.reason = x.reason;
    	break;
	}
} if(x is Object || x is Function) {
	// 如果x是一个对象或者函数，这里主要针对thenable
  try {
  	then = x.then // [5]
  } catch(e) {
  	// 如果x没有then方法，或者获取x.then的时候报错
  	reject promise with e;
  }
  
  if(then is Function) {
  	try{
      then.call(x, resolvePromise, rejectPromise);
      // 如果then方法执行过程中，调用了resolvePromise(y)，则执行[[Resolve]](promise, y)
      // 如果then方法执行过程中，调用了rejectPromise(e)，则reject promise with e
      // 如果resolvePromise和rejectPromise都被调用，多单个方法被多次调用，以第一个调用为主，之后都被忽略
  	} catch(e) {
  		// 如果已经调用过resolveP或rejectPromise，忽略e
  		reject promise with e;
  	}
  } else {
  	// 如果then不是一个方法，即x.then不是一个方法
  	fulfill promise with x;
  }
} else {
	// 如果x不是function
	fulfill promise with x;
}
```

如果参与循环thenable链的thenable解决了promise，那么**\[[Resolve]](promise, thenable)**会产生无限递归，这种情况下的拒绝可以自行实现，但是不是必须的【6】。如果x是这样的object：

```js
let x = {
  then(resolve, reject) {
    resolve(x); // 会产生[[Resolve]](promise, x)，从而产生无限递归
  }
}
new Promise(resolve => resolve(x));
Promise.resovle(x) // 真的会引起死循环
```

### 备注

1. 解释一下“平台代码”：引擎、环境和promise实现的代码。实际上，因为promise代表一个异步操作的最终值，所以onFulfilled和onRejected都是异步执行的，在当前执行栈的末尾，下次事件循环的开始，也就是MicroTask微任务级别。

   > This can be implemented with either a “macro-task” mechanism such as setTimeout or setImmediate, or with a “micro-task” mechanism such as MutationObserver or process.nextTick. 

   注意：这里列举了setTimeout和setImmediate（仅node）为宏任务，除此之外setInterval，requestAnimationFrame（仅浏览器）和I/O事件触发回调，都算作宏任务。同时还列举了MutationObserver（仅浏览器）和process.nextTick（仅node）为微任务，Promise.then/catch/finally也加入微任务大军。（finally不要求一定实现）

2. 什么叫**called as functions**，即以函数的身份执行。我们说“函数”是不依赖调用对象的function，而“方法”应当是依赖调用对象的，如浏览器的全局函数atob，Math类的静态方法random。onFulfilled和onRejected执行时，this一般指向global，严格模式下为undefined。

3. **then**方法返回的promise在一定条件下可以和原promise相同，即promise2 === promise1，只是要指明条件。

4. 一般来说，“if x is a promise”是指x是当前实现下的promise实例，所以“adopt the state”也是在特定于实现的前提下采用状态。

5. “let then = x.then”是指先保存x.then的引用，然后测试该引用，再调用该引用，主要是为了防止多次访问x.then属性。

   >  Such precautions are important for ensuring consistency in the face of an accessor property, whose value could change between retrievals.

   这句话的含义并不明确，大概是说保存一个引用，可以防止访问者属性在检索（也就是取用）过程中的变化，暂时没有想到这种变化是怎样的。

6. 对于x.then会产生无限递归的情况，promise实现不能对thenable链设定随意的深度限制，并假设超过此限制的递归是无线递归。只有在promise和x为同一个引用的时候，才能认为TypeError（即真正的死循环）。如果是由不同的thenable组成的，那无穷递归是正确的行为。