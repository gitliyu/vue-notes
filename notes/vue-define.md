# Vue实例化过程
首先来搞清楚Vue对象到底是什么，也就是说当我们使用import引入Vue时，代码的执行过程以及我们拿到的Vue对象是如何定义的，接下来一步步的翻下源码。

参考： ['vue2.0-source'](https://github.com/liutao/vue2.0-source) ['Vue源码之new Vue'](https://blog.csdn.net/yayayayaya_/article/details/80885473) ['learnVue'](https://github.com/answershuto/learnVue)
### 入口文件
首先第一步，我们打开`package.json`文件，找到以下代码
```json
{
  "scripts": {
    "dev": "rollup -w -c scripts/config.js --environment TARGET:web-full-dev"
  }
}
```
也就是说在执行`npm run dev`命令后会使用['Rollup'](https://www.rollupjs.com/guide/zh)进行打包。
接下来我们来到打包的配置文件`scripts/config.js`查找到以下代码
```javascript
const builds = {
  // Runtime+compiler development build (Browser)
  'web-full-dev': {
    entry: resolve('web/entry-runtime-with-compiler.js'), // 入口文件
    dest: resolve('dist/vue.js'),             // 输出文件
    format: 'umd',
    env: 'development',
    alias: { he: './entity-decoder' },
    banner
  },
  ……
}
```
可以看到文件来源是`web/entry-runtime-with-compiler.js`，这里使用的`resolve`方法是封装过的，通过查看源码和`scripts/alias.js`中对于别名的设置，可以很容易的找到，Vue实际入口文件的地址为
```
src/platforms/web/entry-runtime-with-compiler.js
```
接下来我们一步一步的来找

src/platforms/web/entry-runtime-with-compiler.js
```javascript
import config from 'core/config'
import { warn, cached } from 'core/util/index'
import { mark, measure } from 'core/util/perf'

import Vue from './runtime/index'
```
src/platforms/web/runtime/index.js
```javascript
import Vue from 'core/index'
```
src/core/index.js
```javascript
// 定义Vue 核心方法
import Vue from './instance/index'
// 初始化全局Api   
import { initGlobalAPI } from './global-api/index'  
```
### Vue的定义
找到这一步，就可以看到两个很关键的地方， 首先打开`src/core/instance/index.js`，来看一下以下代码，以及我写的一些注释
```javascript
import { initMixin } from './init'
import { stateMixin } from './state'
import { renderMixin } from './render'
import { eventsMixin } from './events'
import { lifecycleMixin } from './lifecycle'
import { warn } from '../util/index'

function Vue (options) {
  if (process.env.NODE_ENV !== 'production' &&
    !(this instanceof Vue)
  ) {
    warn('Vue is a constructor and should be called with the `new` keyword')
  }
  // new Vue() 实例化时首先调用的方法，在initMixin方法中定义
  this._init(options)
}

// 初始化的入口
initMixin(Vue)
// 数据绑定的核心方法，$set、$delete、$watch等
stateMixin(Vue)
// 事件的核心方法，$on、$once、$off、$emit
eventsMixin(Vue)
// 生命周期的核心方法
lifecycleMixin(Vue)
// 渲染的核心方法，用来生成render函数以及VNode
renderMixin(Vue)

export default Vue

```
可以很清楚的看到，Vue实际上是使用构造函数创建的一个对象方法，之后使用了类似`initMixin`之类的方法来初始化一下功能，挂载了很多实例化之后调用的方法，接下来分别介绍一下。

#### initMixin
打开`src/core/instance/init.js`，找到`initMixin`方法的定义

首先可以看到，在Vue的原型上定义了`_init`方法，在构造时调用这个方法来初始化Vue实例，并且定义了一些像`_uid`，`_isVue`这样的变量
```javascript
export function initMixin (Vue: Class<Component>) {
  Vue.prototype._init = function (options?: Object) {
    const vm: Component = this
    // 自增的id
    vm._uid = uid++

    let startTag, endTag
    /* istanbul ignore if */
    if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
      startTag = `vue-perf-start:${vm._uid}`
      endTag = `vue-perf-end:${vm._uid}`
      mark(startTag)
    }

    // 一个防止vm实例自身被观察的标志位
    vm._isVue = true
```
接下来合并了`options`配置，这样就可以通过`vm.$options.el`等访问到创建实例时传入的`option`
```javascript
// merge options
if (options && options._isComponent) {
  // optimize internal component instantiation
  // since dynamic options merging is pretty slow, and none of the
  // internal component options needs special treatment.
  initInternalComponent(vm, options)
} else {
  vm.$options = mergeOptions(
    resolveConstructorOptions(vm.constructor),
    options || {},
    vm
  )
}
```
接着，是对vue实例进行一些初始化，如生命周期，事件中心，渲染等等，可以看到这里的init方法和`instance.js`中的mixin方法都是对应的，在这里初始化之后再对实例化的对象进行方法的挂载。
```javascript
// 初始化生命周期
initLifecycle(vm)
// 初始化事件
initEvents(vm)
// 初始化render
initRender(vm)
// 调用beforeCreate钩子函数并且触发beforeCreate钩子事件
callHook(vm, 'beforeCreate')
initInjections(vm) // resolve injections before data/props
// 初始化props、methods、data、computed与watch
initState(vm)
initProvide(vm) // resolve provide after data/props
// 调用created钩子函数并且触发created钩子事件
callHook(vm, 'created')
```
最后是判断vm实例是否存在`vm.$options.el`，存在的话就将vm挂载到这个dom节点上，完成渲染，对于`$mount`在['生命周期'](https://github.com/gitliyu/vue-notes/blob/master/notes/vue-lifecycle.md)有介绍
```javascript
if (vm.$options.el) {
  vm.$mount(vm.$options.el)
}
```
#### initState
```javascript
export function initState (vm: Component) {
  vm._watchers = []
  const opts = vm.$options
  // 初始化props
  if (opts.props) initProps(vm, opts.props)
  // 初始化方法
  if (opts.methods) initMethods(vm, opts.methods)
  // 初始化data
  if (opts.data) {
    initData(vm)
  } else {
    // 该组件没有data的时候绑定一个空对象
    observe(vm._data = {}, true /* asRootData */)
  }
  // 初始化computed
  if (opts.computed) initComputed(vm, opts.computed)
  // 初始化watchers
  if (opts.watch) initWatch(vm, opts.watch)
}
```
对Vue对象数据的一下初始化操作

#### stateMixin
相比`initMixin`就显得比较简单了，可以很清楚的看到做的几件事情
```javascript
export function stateMixin (Vue: Class<Component>) {
  const dataDef = {}
  // _data 被observe的存储data数据的对象
  dataDef.get = function () { return this._data }
  const propsDef = {}
  // _props 被observe的存储prop数据的对象
  propsDef.get = function () { return this._props }
  ……
  // Vue 实例代理了对其 data 和 props 对象属性的访问
  Object.defineProperty(Vue.prototype, '$data', dataDef)
  Object.defineProperty(Vue.prototype, '$props', propsDef)

  // 将 Vue.set和Vue.delete定义在原型上
  Vue.prototype.$set = set
  Vue.prototype.$delete = del

  // 为对象建立观察者
  Vue.prototype.$watch = function (
    expOrFn: string | Function,
    cb: Function,
    options?: Object
  ): Function {
    const vm: Component = this
    options = options || {}
    options.user = true
    const watcher = new Watcher(vm, expOrFn, cb, options)
    // 有immediate参数的时候会立即执行
    if (options.immediate) {
      cb.call(vm, watcher.value)
    }
    // 返回一个取消观察函数，用来停止触发回调
    return function unwatchFn () {
      // 将自身从所有依赖收集订阅列表删除
      watcher.teardown()
    }
  }
}
```
完成了对`$data`, `$props`, `$set`, `$delete`和`$watch`的定义，将数据代理在`vm`上，在['Observer与响应式数据'](https://github.com/gitliyu/vue-notes/blob/master/notes/vue-observer.md)有这部分的介绍

#### eventsMixin
`eventsMixin`方法也是在Vue原型上简单粗暴的定义了以下事件
- $on 在vm实例上绑定事件方法
- $once 注册一个只执行一次的事件方法
- $off 注销一个事件，如果不传参则注销所有事件，如果只传event名则注销该event下的所有方法
- $emit 触发一个事件方法

#### initLifecycle
```javascript
export function initLifecycle (vm: Component) {
  const options = vm.$options

  // locate first non-abstract parent
  let parent = options.parent
  if (parent && !options.abstract) {
    while (parent.$options.abstract && parent.$parent) {
      parent = parent.$parent
    }
    parent.$children.push(vm)
  }

  vm.$parent = parent
  vm.$root = parent ? parent.$root : vm

  vm.$children = []
  vm.$refs = {}

  vm._watcher = null
  vm._inactive = null
  vm._directInactive = false
  vm._isMounted = false
  vm._isDestroyed = false
  vm._isBeingDestroyed = false
}
```
方法内容比较简单，定义了$parent、$root、$children属性，并且添加了一下生命周期的标识变量

#### initRender
```javascript
// 初始化render
export function initRender (vm: Component) {
  vm._vnode = null // the root of the child tree
  vm._staticTrees = null
  // 父树中的占位符节点
  const parentVnode = vm.$vnode = vm.$options._parentVnode 
  const renderContext = parentVnode && parentVnode.context
  vm.$slots = resolveSlots(vm.$options._renderChildren, renderContext)
  vm.$scopedSlots = emptyObject

  // 注册createElement方法
  vm._c = (a, b, c, d) => createElement(vm, a, b, c, d, false)
  vm.$createElement = (a, b, c, d) => createElement(vm, a, b, c, d, true)
}
```
这里给vm添加了一些虚拟dom、slot等相关的属性和方法，并将`createElement`方法注册在了vm上，`createElement`用于创建虚拟dom节点`VNode`，会在['数据驱动视图的方法'](https://github.com/gitliyu/vue-notes/blob/master/notes/vue-render.md)进行详细介绍

#### renderMixin
主要做了两件事，绑定了$nextTick事件以及VNode相关的定义
```javascript
export function renderMixin (Vue: Class<Component>) {
  // install runtime convenience helpers
  installRenderHelpers(Vue.prototype)

  Vue.prototype.$nextTick = function (fn: Function) {
    return nextTick(fn, this)
  }

  // _render渲染函数，返回一个VNode节点
  Vue.prototype._render = function (): VNode {}
}
```
关于`_render`渲染函数会在['数据驱动视图的方法'](https://github.com/gitliyu/vue-notes/blob/master/notes/vue-render.md)进行详细介绍

> 总结： 在Vue实例化的过程中，主要完成了对配置的合并，对生命周期、事件中心、`data`、`props`、`computed`、渲染方法等得初始化，并通过对`el`的判断完成实例在DOM上的挂载