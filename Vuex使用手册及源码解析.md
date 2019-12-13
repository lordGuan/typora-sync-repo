## Vuex是什么

> Vuex 是一个专为 Vue.js 应用程序开发的**状态管理模式**。

集中存储（states，getters）并管理（mutations，actions）应用中组件所使用的状态。

这个所谓的状态到底是什么？状态的一般定义--人或事物表现出来的状况和形态。Vue.js应用推荐将组件通过<template></template>，<script></script>和<style></style>三个部分进行组织。分别对应你要维护的视图、逻辑和样式。我们这里着重看视图和逻辑部分，视图用于显示数据和接受用户输入，逻辑部分则负责处理数据和响应输入。Script部分通常导出一个用于初始化Vue实例的options对象，包含name，data和methods等各种属性。其中data属性（函数）返回一个数据集合（普通对象），作为当前组件需要使用和维护的数据集合。这个集合中的数据，有的用于界面的显示，有的用于保存用户的输入，有的只用在逻辑处理过程中。这些数据可以认为是这个组件独有的（不考虑传递出去的情况），所以在某一时刻，这些数据所组成的集合可以认为是这个组件的一个状态，每个字段通过特定的含义描述，可以表达组件当前的状态，而外界也通过视图上绑定的值来观测这个组件，因此最直观的理解，状态就是组件内部的数据。

那么扩大一下，一个Vue.js应用包含非常多的组件，大多都是围绕显示数据和处理数据的，那么从层级视角来看，应用包含组件，组件包含状态，应用可以认为是分区维护大量数据的管理者。有的组件会随着外界的输入或者随着时间变化，表现出不同的形态，也有的组件始终不会变化，发生变化的驱动因素，就是组件内部的数据变化，数据变化导致组件形态产生变化。

每个组件状态管理在封闭的情况下，可以简单看成是：data渲染在view上，methods操作data，view上触发methods。而一个应用中，不同组件维护的状态，总会遇到相同含义，或者相同变化的状态（比如多语言，主题颜色，全局用户名等），这些状态分散在不同的组件中，可能显示不同，也可能有不同的操作，但是他们在逻辑上总是保持一致的。这个时候就需要统一的状态管理，多个用到相同数据的组件，在管理下就能统一表现特定的状态。

Vuex可以理解成一个全局data，将组件共用的状态，从零散的组件中抽象出来进行统一管理。或者说Vuex像是一个特殊的组件，没有视图，但是有数据层和逻辑层，需要表现出来的状态插入到需要的组件中进行展示，组件需要操作状态需要调用Vuex的逻辑层来进行修改。这个流程可以表达为：同一个数据影响多个组件的状态，多个组件的动作变更同一个数据。

需要统一的数据管理主要是为了状态的复用，单向数据流导致数据由父级流向子级的过程是相对复杂的，Vuex将所有组件视为同层次的组件，需要即可注入，都作为Vuex的单一下级组件。同时Vuex也保留了单向数据流的特性，组件通过提交mutation或触发action来让Vuex接受到自己的诉求，数据在哪里定义就在哪里修改，否则将不知道是谁变更的数据。

![Vuex组成](https://vuex.vuejs.org/vuex.png)



所以从极端的情况下，我们可以拥有一个没有自己的data和methods逻辑的组件，状态都依赖Vuex提供，动作都依赖向Vuex发出申请，组件只包含模板进行绑定显示。

至于什么时候你会先用Vuex，大概是在你不想一层一层通过props传递数据，或者在兄弟组件之间相互监听数据时。

## Vuex的核心概念

这里简要提一下：

state：最顶级的数据层，可以认为是被组件复用的状态的集合，data的抽象。组件有data，顶层就有state。mapState辅助函数，减少你写this.\$store.state的机会，可以以你需要的名称将状态映射到组件中。

getter：由state派生出来的状态，computed的抽象。组件有computed，顶层就有getter。组件复用一个状态，往往不是用状态的原始数据，自己的逻辑会对原始数据进行转化，而这个转化的过程有时候也会复用，也就是computed的逻辑被复用，就可以使用getter在顶层就完成state的派生。mapGetters辅助函数，减少写this.\$store.getters的机会。

mutation：是一种统一变更状态的方式。type是一个字符串，用于表达修改的类型，对应的就是handler，包含具体的修改逻辑。而handler可能需要依赖payload进行修改数据，所以就有了**store.commit(type, payload)**的操作格式。mapMutations辅助函数，减少你写this.\$sotre.commit的机会。这里有个习惯，type通常是全局的字符串常量，handler的函数名也使用这个常量进行创建。

action：是mutation的上层管理，可以commit多个mutation，不能直接修改state（可以访问），可以包含异步操作。组件中肯定也有这样的逻辑，把这样的逻辑抽象出来。有了mutation为什么还要action，因为组件逻辑可能需要commit多个mutation，这部分逻辑抽象成了action。逻辑上可以理解为组件dispatch相应的action，由action进行只与state相关，与组件本身逻辑无关的逻辑。action可以包含异步操作，mutation不行，因为mutation就是变更state，针对的就是state。而action相当于mutation的组织者，组织状态转化的流程，因此可以包含异步行为，改变mutation的顺序。mapActions辅助函数，减少你写this.\$store.dispatch的机会。当然了，action中也可以dispatch别的action，从宏观的角度上讲，都是在维护mutation的顺序。action返回Promise是为了让dispatch的逻辑能够知道究竟何时我想要的数据被更新了。

module：复杂的应用状态复杂，某些状态只在某些组件中复用，所以有模块的概念，方便编写时划分逻辑。每个module都有state，mutations

## Vuex的源码分析

### 我们先来看看辅助函数们

由于Vuex的源码不依赖其他包，所以工具方法其实也可以学习一下

#### find(list, f)查找函数

这个函数很有意思：

```js
/**
 * Get the first item that pass the test
 * by second argument function
 *
 * @param {Array} list
 * @param {Function} f
 * @return {*}
 */
export function find (list, f) {
  return list.filter(f)[0]
}
```

和Array.prototype.find方法相同，都是返回满足f条件的第一个元素。但是为什么要用filter，因为find是ES6中的特性，不支持IE浏览器，惊不惊喜，意不意外。

#### deepCopy

深拷贝可以说是数据管理常用的操作了，防止原始数据被污染。

```js
deepCopy (obj, cache = []) {
  // 初等边界判断，同时也是递归边界
  if (obj === null || typeof obj !== 'object') {
    return obj
  }

  // 缓存命中
  const hit = find(cache, c => c.original === obj)
  if (hit) {
    return hit.copy
  }
 
  const copy = Array.isArray(obj) ? [] : {}
  // 置入缓存中
  // 递归调用也可以进行引用
  cache.push({
    original: obj,
    copy
  })

  // 开始对键值进行递归
  Object.keys(obj).forEach(key => {
    copy[key] = deepCopy(obj[key], cache)
  })

  return copy
}
```

这个深拷贝针对Vuex中使用的情况完全足够，状态管理针对的往往都是单纯的状态，而且数据类型的复杂程度不会很高，所以深拷贝的复杂程度不用太细致（相比lodash的cloneDeep）。

#### 其他工具函数

```js
/**
 * forEach for object
 */
export function forEachValue (obj, fn) {
  Object.keys(obj).forEach(key => fn(obj[key], key))
}

// 防止typeof null === 'object'影响操作
export function isObject (obj) {
  return obj !== null && typeof obj === 'object'
}

// 相当于用thenable作为Promise的标准
export function isPromise (val) {
  return val && typeof val.then === 'function'
}

export function assert (condition, msg) {
  if (!condition) throw new Error(`[vuex] ${msg}`)
}

export function partial (fn, arg) {
  return function () {
    return fn(arg)
  }
}
```

#### Store相关工具函数

##### makeLocalContext(store, namespace, path)

创建本地化的dispatch、commit、getters和state。至于这个locallized，我英语实在不咋地，就直译成本地化的。

```js
// store.js
function makeLocalContext (store, namespace, path) {
  const noNamespace = namespace === ''

  // 这个方法会返回一个特殊的对象
  // 如果没有命名空间的话会沿用store.dispatch，注意，这个方法应当是重绑定过的
  const local = {
    dispatch: noNamespace ? store.dispatch : (_type, _payload, _options) => {
      // unifyObjectStyle只是将转换dispatch参数的工具函数，内容不多做介绍
      // 主要是dispatch方法支持两种参数形式，这个方法用于统一化
      const args = unifyObjectStyle(_type, _payload, _options)
      // 这里发现，只读取不操作的变量直接声明成const，会操作的请用let
      const { payload, options } = args
      let { type } = args

      if (!options || !options.root) {
        type = namespace + type
        // 不存在的action将返回
      }

      return store.dispatch(type, payload)
    },

    commit: noNamespace ? store.commit : (_type, _payload, _options) => {
      const args = unifyObjectStyle(_type, _payload, _options)
      const { payload, options } = args
      let { type } = args

      if (!options || !options.root) {
        type = namespace + type
        // 不存在的mutation将返回
      }

      store.commit(type, payload, options)
    }
  }

  // getters and state object必须被懒加载
  Object.defineProperties(local, {
    getters: {
      get: noNamespace
        ? () => store.getters
        : () => makeLocalGetters(store, namespace)
    },
    state: {
      get: () => getNestedState(store.state, path)
    }
  })

  return local
}
```

这个方法将生成一个普通对象，用作模块的context属性，作为上下文使用。方便在子模块中使用高层次的mutation和action等。

### 我们再来看看辅助类

#### Module基类

Module的定义如下：

```ts
export interface Module<S, R> {
  namespaced?: boolean;
  state?: S | (() => S);
  getters?: GetterTree<S, R>;
  actions?: ActionTree<S, R>;
  mutations?: MutationTree<S>;
  modules?: ModuleTree<R>;
}
```

本质结构和Store状态树是一致的，多了一个namespaced字段，用于隔离。构造法如下：

```js
constructor (rawModule, runtime) {
  this.runtime = runtime
  // Store some children item
  this._children = Object.create(null)
  // Store the origin module object which passed by programmer
  this._rawModule = rawModule
  const rawState = rawModule.state

  // Store the origin module's state
  this.state = (typeof rawState === 'function' ? rawState() : rawState) || {}
}
```

rawModule是一个原始的模块定义，包含state，mutations等，Module再此基础上扩展了一个\_children属性用于处理子模块。所以Store本身相当于跟模块。对child的操作包括addChild、removeChild、getChild和forEachChild。提供一个update方法，接受原始模块定义，但只能更新namespaced属性、actions、mutations和getters。

#### ModuleCollection基类

ModuleCollection依赖原始跟模块定义进行构造，然后注册跟模块：

```js
constructor (rawRootModule) {
  // 跟模块路径为空
  this.register([], rawRootModule, false)
}

register (path, rawModule, runtime = true) {
  if (process.env.NODE_ENV !== 'production') {
    // 该方法校验原始模型必须包含getters属性，且包含的值必须是函数
    // 包含mutations属性，包含的值必须是函数
    // 包含actions属性，包含的值必须是对象
    // 这里特殊说明一下actions属性，值可以是函数，或者是，包含一个值为函数的handler属性的对象
    assertRawModule(path, rawModule)
  }

  // 构造根模块
  const newModule = new Module(rawModule, runtime)
  if (path.length === 0) {
    this.root = newModule
  } else {
    // get方法根据path元素reduce出子元素
    const parent = this.get(path.slice(0, -1))
    // 拿到父级元素来添加子模块，这是嵌套注册
    parent.addChild(path[path.length - 1], newModule)
  }

  // 注册嵌套模块
  if (rawModule.modules) {
    forEachValue(rawModule.modules, (rawChildModule, key) => {
      // 递归的过程中，path逐渐加入模块的名称
      this.register(path.concat(key), rawChildModule, runtime)
    })
  }
}
```

ModuleCollection实例方法get是一个按路径获取子模块的方法：

```js
get (path) {
  return path.reduce((module, key) => {
    return module.getChild(key)
  }, this.root)
}

getNamespace (path) {
  let module = this.root
  return path.reduce((namespace, key) => {
    module = module.getChild(key)
    return namespace + (module.namespaced ? key + '/' : '')
  }, '')
}
```

path参数是一个字符串数组，可以认为是*模块树*中描述某一个节点的路径。这里使用reduce函数，跟模块作为起始状态，从模块拿取对应的子模块，然后在这个子模块上继续拿取，按照节点的路径在*模块树*中获取对应的节点。

相应的getNamespace也是通过reduce函数来将描述*模块树*路径转换成一个可读的字符串，只是要根据模块的namespaced属性来进行划分，在命名空间上是否应当算作独立的一层。

ModuleCollection作为顶层模块基类，提供一个update方法，用于更新其下属的模块。由于子模块是Module类的实例，所以直接调用相应的update方法即可。如果新模块也包含嵌套模块，就要进行递归调用，所以update方法的实际逻辑是单独出一个函数完成的。

```js
update (rawRootModule) {
  update([], this.root, rawRootModule)
}

function update (path, targetModule, newModule) {
  if (process.env.NODE_ENV !== 'production') {
    assertRawModule(path, newModule)
  }

  // 直接更新模块，相当于更新模块树局部的根节点
  targetModule.update(newModule)

  // 如果用于更新的模块包含子模块，在进行递归更新，相当于用新的“小树枝”更新原来模块树中的局部
  if (newModule.modules) {
    for (const key in newModule.modules) {
      if (!targetModule.getChild(key)) {
        // ...这里有一段关于热重载的警告
        return
      }
      update(
        path.concat(key),
        targetModule.getChild(key),
        newModule.modules[key]
      )
    }
  }
}
```



### Vue.use(Vuex)从装载开始

Vuex的install方法，仅允许绑定一次，通过Vuex内部的一个全局常量维持，防止多次挂在状态树，保证唯一性。然后对Vue执行混入。applyMixin对Vue的*\_init*过程进行了处理，将 **vuexInit**加入到Vue的初始化过程中。

```js
function vuexInit () {
    const options = this.$options // 因为是在Vue初始化流程中进行，所以这是Vue实例的配置项
    // store injection
    if (options.store) {
      // 提供的store可能是创建函数，也可能是创建好的实例，挂载到Vue实例的$store属性上
      this.$store = typeof options.store === 'function'
        ? options.store()
        : options.store
    } else if (options.parent && options.parent.$store) {
      // 对于子组件而言，从父级将$store继承下来
      // 像elementUI中，使用服务式调起dialog，是无法从根组件上继承下来$store，声明式的可以
      this.$store = options.parent.$store
    }
  }
```

**vuexInit**初始化流程如上述代码，由于该方法被放到了Vue实例\_init过程中，所以this都指向响应的Vue实例。

### new Vuex.Store()创建状态树

**Store**类是Vuex的核心基类，用于创建状态树实例。Vuex支持提供一个Store实例作为单一状态树，也支持提供一个创建函数来创建单一状态树，后者可以保证提前创建好的Store实例被Vuex以外的地方操作。（如果还没有使用Vue.use，Vuex会尝试装在到window.Vue上）。

> 这里额外提一下Store的构造函数，其中包含了三个限制：1.必须要装在到Vue上，即调用过Vue.use(Vuex)；2.要支持Promise；3.只能通过new Store()进行使用（这也是一般的构造法需要提示的）。

接下来会初始化一堆内置状态，其中有一个**\_watcherVm**属性是一个Vue实例，**\_modules**是ModuleCollection实例。并且空对象的创建都是通过Object.create(null)进行的。

```js
// bind commit and dispatch to self
const store = this
const { dispatch, commit } = this
this.dispatch = function boundDispatch (type, payload) {
  return dispatch.call(store, type, payload)
}
this.commit = function boundCommit (type, payload, options) {
  return commit.call(store, type, payload, options)
}
```

这段代码比较有意思，将dispatch方法和commit方法强行和实例进行绑定，这样在action中使用通过参数传入的commit和dispatch能够确保作用在状态树上。

#### installModule (store, rootState, path, module, hot)装载模块

在重绑定了dispatch和commit之后，就开始核心阶段，之前通过new Store(options)的options来创建ModuleCollection实例，相当于已经有了模块树，然后installModule对状态树进行接管

```js
// 初始化跟模块.
// 递归注册子模块
// 并收集getters
installModule(this, state, [], this._modules.root)
```

这里说明一下，ModuleCollection是一个根据用户配置，产生的一个封装后的模块树实体，提供了针对模块的一些操作。而这里installModule是让Vuex能够针对模块树中每个节点的状态等进行管理。

```js
function installModule (store, rootState, path, module, hot) {
  const isRoot = !path.length
  const namespace = store._modules.getNamespace(path)

  // store维护了一个命名空间的map，如果设定了namespaced属性，则需要特殊记录一下
  // 命名空间其实就是该模块在模块树中的路径描述
  if (module.namespaced) {
    // ...这里有一段关于重名模块的报错
    store._modulesNamespaceMap[namespace] = module
  }

  // set state
  if (!isRoot && !hot) {
    // getNestedState仍然是一个按照path进行reduce获取正要操作节点的父节点的state
    const parentState = getNestedState(rootState, path.slice(0, -1))
    const moduleName = path[path.length - 1]
    // _withCommit表示这个操作类似commit操作，所以需要操作_commiting锁属性
    // _commiting锁属性表达了可信赖的state操作，如果_commiting为false，则不能直接对state进行操作
    store._withCommit(() => {
      // ...这里有一段关于同名模块覆盖的警告
      // 这里借助Vue.set方法来注册响应式的state
      Vue.set(parentState, moduleName, module.state)
    })
  }

  // context属性非常明确，就是上下文属性，可以用子模块使用高层次模块的内容
  const local = module.context = makeLocalContext(store, namespace, path)

  // 然后开始注册mutation、action、getter
  module.forEachMutation((mutation, key) => {
    const namespacedType = namespace + key
    registerMutation(store, namespacedType, mutation, local)
  })

  module.forEachAction((action, key) => {
    const type = action.root ? key : namespace + key
    const handler = action.handler || action
    registerAction(store, type, handler, local)
  })

  module.forEachGetter((getter, key) => {
    const namespacedType = namespace + key
    registerGetter(store, namespacedType, getter, local)
  })

  module.forEachChild((child, key) => {
    // 递归完成模块树的安装操作
    installModule(store, rootState, path.concat(key), child, hot)
  })
}
```

可以看到在注册mutation、action和getter时，是传入了local，即上下文。mutation对应commit，commit是允许在命名空间模块中提交根模块的mutation；action对应dispatch，dispatch也是允许在命名空间模块中分发根模块的action；而getter是基于state派生的计算属性，在命名空间模块定义的时候，getter可能会有些不一样：

```ts
export type Getter<S, R> = (state: S, getters: any, rootState: R, rootGetters: any) => any;
```

可见getter是可以依赖当前模块的state、当前模块的getters、根模块的state和根模块的getters，所以也是需要上下文的。那么下面我们来看一下是如何注册的。

##### registerMutation、registerAction、registerGetter

注册mutation的过程会对Store实例的私有属性\_mutation进行操作，按照设定的mutation生成对应的handler。

```js
// store.js
function registerMutation (store, type, handler, local) {
  const entry = store._mutations[type] || (store._mutations[type] = [])
  entry.push(function wrappedMutationHandler (payload) {
    handler.call(store, local.state, payload)
  })
}
```

所有模块的mutation等都是注册在顶层store的，这里也可以看到是挂在store.\_muttations属性上的，因此在没有命名空间的限制下，mutation名称是可能重复的，所以这里对某个type的mutation将生成一个数组来存放handler函数。

注册action和注册mutation类似，只是actionHandler不太相同，根据action的使用，允许返回一个结果，这个结果可以是Promise。由于handler是用户定义的，为了成功包装，需要将其返回的Promise直接resolve出去。而且action中是有devtoolHook的，针对handler返回的被reject的Promise来进行catch，暂时先不关注这些。

```js
// store.js
function registerAction (store, type, handler, local) {
  const entry = store._actions[type] || (store._actions[type] = [])
  entry.push(function wrappedActionHandler (payload) {
    // 这里可以看到定义action时所用到的参数
    // action可以分发其他action，可以提交mutation，可以访问各种state和getter
    let res = handler.call(store, {
      dispatch: local.dispatch,
      commit: local.commit,
      getters: local.getters,
      state: local.state,
      rootGetters: store.getters,
      rootState: store.state
    }, payload)
    if (!isPromise(res)) {
      res = Promise.resolve(res)
    }
    if (store._devtoolHook) {
      return res.catch(err => {
        store._devtoolHook.emit('vuex:error', err)
        throw err
      })
    } else {
      return res
    }
  })
}
```

注册getter也是差不多，store维护一个私有属性\_wrappedGetters，按照设定的getters包装一层。这里不向mutation那样是数组存放，getter的注册在没有命名空间的情况下是不允许重名的。

```js
function registerGetter (store, type, rawGetter, local) {
  if (store._wrappedGetters[type]) {
    // ...这里有一个关于重名getter的报错
    return
  }
  // 这里依然使用封装后的函数，防止用于配置Store的原始数据遭到滥用
  store._wrappedGetters[type] = function wrappedGetter (store) {
    return rawGetter(
      local.state, // local state
      local.getters, // local getters
      store.state, // root state
      store.getters // root getters
    )
  }
}
```

#### resetStoreVM (store, state, hot)初始化或重置store vm

装载完模块并注册了各种信息后，要对store进行响应式处理，这一步才真正将getter处理成响应式的，相当于作为订阅者去与相关数据进行绑定。

```js
// store.js
function resetStoreVM (store, state, hot) {
  const oldVm = store._vm

  // 对外暴露的getters对象
  store.getters = {}
  // 重置getters缓存
  store._makeLocalGettersCache = Object.create(null)
  // 依赖刚才registerGetter的结果_wrappedGetters
  const wrappedGetters = store._wrappedGetters
  const computed = {}
  forEachValue(wrappedGetters, (fn, key) => {
    // 将store和getter函数内联起来，创造保留了旧vm的闭包
    computed[key] = partial(fn, store)
    // 把getters的访问移交到_vm上
    Object.defineProperty(store.getters, key, {
      get: () => store._vm[key],
      enumerable: true // for local getters
    })
  })

  // 神奇的一批，store的_vm属性代表其实例，没错是一个Vue实例
  // 临时修改了Vue的silent配置，稍后就归还
  // 所以还真是state对应data，getters对应computed
  const silent = Vue.config.silent
  Vue.config.silent = true
  store._vm = new Vue({
    data: {
      $$state: state
    },
    computed
  })
  Vue.config.silent = silent

  // 启用严格模式，这里比较简单
  // 为_vm附加一个watch，$$state一旦发生变化，就会校验commiting锁，非法操作将报错
  if (store.strict) {
    enableStrictMode(store)
  }

  // 有oldVm是重置，没有的话初始化_vm已经完事儿了
  if (oldVm) {
    if (hot) {
      // dispatch changes in all subscribed watchers
      // to force getter re-evaluation for hot reloading.
      store._withCommit(() => {
        oldVm._data.$$state = null
      })
    }
    // 销毁旧的实例
    Vue.nextTick(() => oldVm.$destroy())
  }
}
```

resetStoreVM这一步可以看出，vuex使用的状态树，或者说状态容器，可以认为是一个封装的Vue组件，没有视图，但是提供数据的相应式使用。

整个store实例的构造方法的核心操作主要是以上几个方法，还有遍历传入的plugins属性加载插件，和根据需求配置devtoolPlugin，这两个功能的实现比较简单，有兴趣自己找来看就行。至此，作为顶级特殊组件的store就随着Vue的初始化完成构造，并挂到了实例的\$store属性上，接下来我们看看使用过程中，vuex是如何工作的。

### 使用State

先从最简单的使用开始，我们在组件中可以直接通过this.\$store.state.xxx来访问根模块下的xxx状态。\$store属性是Store的实例，初始化这个实例的时候，并没有将配置中的各种状态直接作为这个实例的属性进行初始化到实例属性，所以我们来接着看Store基类的其他实例方法定义。

```js
get state () {
  return this._vm._data.$$state
}

set state (v) {
  if (process.env.NODE_ENV !== 'production') {
    assert(false, `use store.replaceState() to explicit replace store state.`)
  }
}
```

this.\$store.state的访问和操作，是基于setter和getter，所以我们没有直接的看到这个属性。如果我们将this.\$store.state.xxx和视图进行了绑定，在这个状态发生变化的同时，视图也会发生变化，这是因为getter访问器，将内置的\_vm中被响应式处理的状态置翻出来，这会使得组件中的对应部分直接成为这个状态的一个监听者，会被Vue收集依赖机制所处理。不同层级的组件依赖store中的状态，视图层或功能层面可能属于不同层级，但是从数据层面它们居然是平级的。这里的setter要对直接的state操作进行屏蔽。

#### 工具函数mapState

书写大量的this.\$store.state实在是有些蠢，虽然你可以在你的组件里写很多computed来将你需要的状态弄出来，甚至赋予一个更有实际意义的名字，但是从代码规模上可能还是比较庞大的。

这里先说明一下normalizeNamespace和normalizeMap方法。normalizeNamespace方法主要是将目标函数fn进行一些修饰，对fn所需的namespace参数和map参数进行预处理。normalizeMap则是一个数据适配器函数，格式化用户的参数为可用的结构。

```js
// helpers.js
function normalizeNamespace (fn) {
  return (namespace, map) => {
    if (typeof namespace !== 'string') {
      map = namespace
      namespace = ''
    } else if (namespace.charAt(namespace.length - 1) !== '/') {
      namespace += '/'
    }
    return fn(namespace, map)
  }
}

/**
 * normalizeMap([1, 2, 3]) => [ { key: 1, val: 1 }, { key: 2, val: 2 }, { key: 3, val: 3 } ]
 * normalizeMap({a: 1, b: 2, c: 3}) => [ { key: 'a', val: 1 }, { key: 'b', val: 2 }, { key: 'c', val: 3 } ]
 */
function normalizeMap (map) {
  if (!isValidMap(map)) {
    return []
  }
  return Array.isArray(map)
    ? map.map(key => ({ key, val: key }))
    : Object.keys(map).map(key => ({ key, val: map[key] }))
}
```

我们在使用mapState工具函数时，我们要将这个函数的返回值结构到组件的computed属性中，那么说明这个函数将返回对象，被打散后成为computed值，神奇吗，还真是把你写this.\$store.state的功夫给省了。当然了，还有一些其他的操作，让你不用操心命名空间等。

```js
// helpers.js
export const mapState = normalizeNamespace((namespace, states) => {
  const res = {}
  // ...这里有个关于参数校验的提示
  normalizeMap(states).forEach(({ key, val }) => {
    // res显然就成了computed的部分键值对，注意这里用的是function还不是箭头函数
    // 在组件中打散后，this才能正确指向vue实例，不需要额外的环境就能拿起store实例
    res[key] = function mappedState () {
      let state = this.$store.state
      let getters = this.$store.getters
      if (namespace) {
        // 相当于this.$store._modulesNamespaceMap[namespace]
        // 只是有一个额外的提示，命名空间不存在
        const module = getModuleByNamespace(this.$store, 'mapState', namespace)
        if (!module) {
          return
        }
        state = module.context.state
        getters = module.context.getters
      }
      // 特别注意：这里val.call的形式，其实是为了让你能用到组件局部的状态
      // 试想：如果这里直接是val(state, getters)会发生什么
      return typeof val === 'function'
        ? val.call(this, state, getters)
        : state[val]
    }
    // mark vuex getter for devtools
    res[key].vuex = true
  })
  return res
})
```

在官网的使用教程上也提到了，mapState提供的map方式可以是字符串数组，或一个对象：

```js
export default {
  // ...
  computed: mapState({
    // 箭头函数可使代码更简练，不可以使用this，否则会相当于在window对象上取字段
    count: state => state.count,

    // 传字符串参数 'count' 等同于 `state => state.count`
    countAlias: 'count',

    // 为了能够使用 `this` 获取局部状态，必须使用常规函数
    countPlusLocalState (state) {
      return state.count + this.localCount
    }
  })
}
```

组件所需的计算值名称如果和state中的状态名称相同，可以使用mapState(['count'])这样的形式，我们着重看一下对象式使用。给mapState提供的这个对象，可以看做是映射关系的集合，这个对象的key是组件需要的计算属性名，而value可以是字符串，箭头函数或普通函数，用于告知vuex按照value指明的“关系”进行处理。使用箭头函数，是不可以使用this的，由于箭头函数特殊性。我们直接关注mapState的返回结果，可以将上述代码看成这样：

```js
export default {
  // ...
  computed: {
    count: function mappedState () {
      // 这里简单了一些内容
      let state = this.$store.state;
      let mapFunction = state => state.count
      return mapFunction.call(this, state)
    },
    countAlias: : function mappedState () {
      let state = this.$store.state;
      return state['count']
    },
    countPlusLocalState: function mappedState () {
      let state = this.$store.state;
      let mapFunction = function (state) {
        return state.count + this.localCount
      }
      return mapFunction.call(this, state)
    }
  }
}
```

所以1.state缓存了this.state；2.绑定了this使得常规函数定义的映射关系可以取到组件局部的值。

这里额外说一下，上面代码export的对象，相当于初始化一个Vue实例用的options，初始化Vue实例的时候，options中的computed属性会被收集依赖，并通过defineProperty挂到实例上，所以在用computed计算属性时是直接使用的。当然了，computed中定义的函数用于指明计算关系，必须是常规函数，保留正确的上下文环境。computed和mapState的结果都是常规函数，最内层又使用call进行调用，这是保证this可用的大前提。这一段代码充分体现了上下文环境在JavaScript程序中的重要性，分离度很高的代码，也可以通过合理调整上下文，来达到协同工作的目的。

### 使用getters

和state不同，getters并没有对应的访问器，是一个直接挂载到实例上的属性，那么这个属性在哪里挂载的，我们就要追溯到resetStoreVM，毕竟getters是state的一个衍生，其响应式依赖于state的响应式，因此这一步肯定会放在vm的操作上。拿出resetStoreVM中相关的代码：

```js
// store.js
// bind store public getters
function resetStoreVM (store, state, hot) {
  const oldVm = store._vm
  store.getters = {}
  // 传说中的getters缓存，用于减少计算量
  store._makeLocalGettersCache = Object.create(null)
  const wrappedGetters = store._wrappedGetters
  const computed = {}
  // 这个属性保存的是创建store时，模块的getters属性
  forEachValue(wrappedGetters, (fn, key) => {
    computed[key] = partial(fn, store)
    // 将store实例上的getters访问，直接给到了内部vm的getters
    Object.defineProperty(store.getters, key, {
      get: () => store._vm[key],
      enumerable: true
    })
  })
  
  const silent = Vue.config.silent
  Vue.config.silent = true
  store._vm = new Vue({
    data: {
      $$state: state
    },
    computed
  })
  Vue.config.silent = silent
  
  // ...其他代码
}
```

这样的就比较清晰了，this.\$store.getters其实直接怼到了store内部\_vm中的计算属性，这和之前分析的state对应data，getters对应computed，基本上就符合了。那这个计算缓存是如何实现的，看一下_makeLocalGettersCache属性的相关代码。

查找\_makeLocalGettersCache可以追溯到registerGetter方法，回顾一下，这个方法其实是生成_wrappedGetters，即在用户配置getters的基础上生成的新函数，用于将本模块和根模块的state和getters传进去，其实就是将整个可用于派生的上下文传到getters里。那么就会产生一个问题，getters依赖state，getters也会依赖getters，不过这里的getters依赖的getters是通过makeLocalGetters生成的可缓存的getters。在依赖getters进行计算时，其实拿到的是\_makeLocalGettersCache中缓存的值，终于追溯到我们需要的东西上了。

```js
// store.js
function makeLocalGetters (store, namespace) {
  // 相当于是否命中缓存
  if (!store._makeLocalGettersCache[namespace]) {
    const gettersProxy = {}
    const splitPos = namespace.length
    Object.keys(store.getters).forEach(type => {
      // 跳过不同命名空间的getters
      if (type.slice(0, splitPos) !== namespace) return

      // extract local getter type
      const localType = type.slice(splitPos)

      // 缓存中保存的是一个代理
      Object.defineProperty(gettersProxy, localType, {
        get: () => store.getters[type],
        enumerable: true
      })
    })
    store._makeLocalGettersCache[namespace] = gettersProxy
  }

  return store._makeLocalGettersCache[namespace]
}
```

再看makeLocalContext的时候，上下文对象local的getters和state都做了处理，当时的注释是为了“延迟加载”，不难想象，因为要向所有的getters注入上下文中的getters环境，如果getters不用，计算就是多余的，因此使用缓存+getters访问器的处理，减少没必要的getters计算。

#### 工具函数mapGetters

整体看来和mapState的思路差不多，resetStoreVM过程中，为实例增加的getters属性，按照map规则被返回出去。

```js
// helpers.js
export const mapGetters = normalizeNamespace((namespace, getters) => {
  const res = {}
  // ...这里有关于映射参数错误的提示
  normalizeMap(getters).forEach(({ key, val }) => {
    // The namespace has been mutated by normalizeNamespace
    val = namespace + val
    res[key] = function mappedGetter () {
      if (namespace && !getModuleByNamespace(this.$store, 'mapGetters', namespace)) {
        return
      }
      if (process.env.NODE_ENV !== 'production' && !(val in this.$store.getters)) {
        console.error(`[vuex] unknown getter: ${val}`)
        return
      }
      return this.$store.getters[val]
    }
    // mark vuex getter for devtools
    res[key].vuex = true
  })
  return res
})
```





