### 从入口文件分析Vue对象及全局API方法
首先来搞清楚Vue对象到底是什么，也就是说当我们使用import引入Vue时，代码的执行过程以及我们拿到的Vue对象是如何定义的，接下来一步步的翻下源码。
首先第一步，我们打开`package.json`文件，找到以下代码
```json
{
  "scripts": {
    "dev": "rollup -w -c scripts/config.js --environment TARGET:web-full-dev",
    ……
  }
}
```
也就是说在执行`npm run dev`命令后会使用['Rollup'](https://www.rollupjs.com/guide/zh)进行打包。
接下来我们来到打包的配置文件`scripts/config.js`查找到以下代码
```javascript
const builds = {
  // Runtime+compiler development build (Browser)
  'web-full-dev': {
    entry: resolve('web/entry-runtime-with-compiler.js'),	// 入口文件
    dest: resolve('dist/vue.js'),							// 输出文件
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
```javascript
// src/platforms/web/entry-runtime-with-compiler.js

import config from 'core/config'
import { warn, cached } from 'core/util/index'
import { mark, measure } from 'core/util/perf'

import Vue from './runtime/index'

// src/platforms/web/runtime/index.js

import Vue from 'core/index'

// src/core/index.js

// 定义Vue 核心方法
import Vue from './instance/index'
// 初始化全局Api		
import { initGlobalAPI } from './global-api/index'	
```
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
  this._init(options)
}

// 初始化的入口
initMixin(Vue)
// 数据绑定的核心方法，$set、$delete、$watch等
stateMixin(Vue)
// 事件的核心方法，$on、$once、$off、$emit等
eventsMixin(Vue)
// 生命周期的核心方法
lifecycleMixin(Vue)
// 渲染的核心方法，用来生成render函数以及VNode
renderMixin(Vue)

export default Vue

```
可以很清楚的看到，Vue实际上是使用构造函数创建的一个对象方法，之后使用了类似`initMixin`之类的方法来初始化一下功能，挂载了很多实例化之后调用的方法。
接下来看一下`initGlobalAPI`这个方法，打开`src/core/global-api/index.js`文件，下面贴一下主要代码
```javascript
export function initGlobalAPI (Vue: GlobalAPI) {
  // config
  const configDef = {}
  configDef.get = () => config
  if (process.env.NODE_ENV !== 'production') {
    configDef.set = () => {
      warn(
        'Do not replace the Vue.config object, set individual fields instead.'
      )
    }
  }
  // Vue.config 各种全局配置项
  Object.defineProperty(Vue, 'config', configDef)

  // Vue.util 各种工具函数，还有一些兼容性的标志位
  Vue.util = {
    warn,
    extend,
    mergeOptions,
    defineReactive
  }

  // Vue.set/delete 增加/删除响应式对象属性
  Vue.set = set
  Vue.delete = del
  // Vue.nextTick 下次DOM更新后执行的回调
  Vue.nextTick = nextTick

  Vue.options = Object.create(null)
  ASSET_TYPES.forEach(type => {
    Vue.options[type + 's'] = Object.create(null)
  })

  Vue.options._base = Vue

  extend(Vue.options.components, builtInComponents)

  // Vue.use 安装插件
  initUse(Vue)
  // Vue.mixin 混入实例
  initMixin(Vue)
  // Vue.extend 创建构建器
  initExtend(Vue)
  // Vue.component, Vue.directive, Vue.filter
  initAssetRegisters(Vue)
}
```
可以看到各种全局Api方法的定义，关于方法的使用可以查看['Vue官网'](https://cn.vuejs.org/v2/api/#Vue-extend)