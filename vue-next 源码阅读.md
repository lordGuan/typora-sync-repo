## 前言



## shared

还是从最基础最公共最简单的部分看起

### globalsWhiteslist.ts

常量：globalsWhiteslist: Set\<string>，关键字、全局方法、全局对象和全局常量等，如Number、undefined和isNaN等

### element.ts

常量：HTMLTagSet: Set\<string>，SVGTagSet: Set\<string>，VoidTagSet: Set\<string>（自封闭标签，如\<br/>），各种tag。

工具函数：isVoidTag: (tag: string): boolean，isHTMLTag: (tag: string): boolean，isSVGTag: (tag: string): boolean，对应的判断函数

### patchFlags.ts 待补全

### index.ts

常量：

```typescript
// 空对象和空数组
export const EMPTY_OBJ: { readonly [key: string]: any } = __DEV__
  ? Object.freeze({})
  : {}
export const EMPTY_ARR: [] = []
```

工具函数：

```typescript
// 什么都不做
const NOOP = () => {}

// 总是返回false
export const NO = () => false

// 用于解析模板，判断是不是“on-”属性
export const isOn = (key: string) => key[0] === 'o' && key[1] === 'n'

// 简单版Object.assign，只用于将额外的对象融入到目标对象
// Object.assign会拷贝访问器
export const extend = <T extends object, U extends object>(
  a: T,
  b: U
): T & U => {
  for (const key in b) {
    ;(a as any)[key] = b[key]
  }
  return a as any
}

// 判断包含属性的工具函数，注意ts的写法
// key is keyof typeof val 这种写法标明这个函数是用于类型约束的
// if(hasOwn(a, b)){...}代码体中才能a[b]这样访问
const hasOwnProperty = Object.prototype.hasOwnProperty
export const hasOwn = (
  val: object,
  key: string | symbol
): key is keyof typeof val => hasOwnProperty.call(val, key)

// 判断类工具函数，注意ts的写法
export const isArray = Array.isArray
export const isFunction = (val: any): val is Function =>
  typeof val === 'function'
export const isString = (val: any): val is string => typeof val === 'string'
export const isSymbol = (val: any): val is symbol => typeof val === 'symbol'
// isObject(null) -> false
export const isObject = (val: any): val is Record<any, any> =>
  val !== null && typeof val === 'object'

// 判断是朴素对象 isPlainObject(null) -> true
export const objectToString = Object.prototype.toString
export const toTypeString = (value: unknown): string =>
  objectToString.call(value)

export const isPlainObject = (val: any): val is object =>
  toTypeString(val) === '[object Object]'

// 字符串工具函数，主要用于模板编译
const vnodeHooksRE = /^vnode/
export const isReservedProp = (key: string): boolean =>
  key === 'key' || key === 'ref' || key === '$once' || vnodeHooksRE.test(key)

const camelizeRE = /-(\w)/g
export const camelize = (str: string): string => {
  return str.replace(camelizeRE, (_, c) => (c ? c.toUpperCase() : ''))
}

const hyphenateRE = /\B([A-Z])/g
export const hyphenate = (str: string): string => {
  return str.replace(hyphenateRE, '-$1').toLowerCase()
}

export const capitalize = (str: string): string => {
  return str.charAt(0).toUpperCase() + str.slice(1)
}
```

## reactive

响应式系统，完全可以单独使用

