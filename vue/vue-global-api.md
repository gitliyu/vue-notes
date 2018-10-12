# Global API
上一篇在入口文件中看到了`initGlobalAPI`方法，那么就来看一下`global-api`这个目录吧，Vue的静态方法大多都是在该文件夹定义的

### 文件结构
首先看一下文件结构，`index.js`里面的引入路径可以很清楚的看出来各个文件的作用，这一篇蛀牙介绍一下`global-api`目录，对于其他位置引入注册的方法暂时不去分析。
```
 |— global-api
   |—asssets.js   // Vue.component, Vue.directive, Vue.filter
   |—extend.js    // Vue.extend
   |—index.js
   |—mixin.js     // Vue.mixin
   |—use.js       // Vue.use
```
### initGlobalAPI
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
### Flow
首先引起我注意的就是在定义时对于参数的检查
```
function initGlobalAPI (Vue: GlobalAPI)
```
对于Vue有些了解的应该都知道Vue源码是使用Flow进行的类型检查，马么我们打开`flow/global-api.js`，看一下对于`GlobalAPI`这个类型的定义
```javascript
declare interface GlobalAPI {
  cid: number;
  options: Object;
  config: Config;
  util: Object;

  extend: (options: Object) => Function;
  set: <T>(target: Object | Array<T>, key: string | number, value: T) => T;
  delete: <T>(target: Object| Array<T>, key: string | number) => void;
  nextTick: (fn: Function, context?: Object) => void | Promise<*>;
  use: (plugin: Function | Object) => void;
  mixin: (mixin: Object) => void;
  compile: (template: string) => { render: Function, staticRenderFns: Array<Function> };

  directive: (id: string, def?: Function | Object) => Function | Object | void;
  component: (id: string, def?: Class<Component> | Object) => Class<Component>;
  filter: (id: string, def?: Function) => Function | void;

  // allow dynamic method registration
  [key: string]: any
};

```

### initUse 
`Vue.use`方法用于安装插件
```javascript
export function initUse (Vue: GlobalAPI) {
  Vue.use = function (plugin: Function | Object) {
    // installedPlugins为已安装插件，检查是否已安装
    const installedPlugins = (this._installedPlugins || (this._installedPlugins = []))
    if (installedPlugins.indexOf(plugin) > -1) {
      return this
    }

    // additional parameters
    // 添加Vue参数
    const args = toArray(arguments, 1)
    args.unshift(this)
    // 如果有install方法，执行install方法，如果传入对象是一个方法，直接执行
    if (typeof plugin.install === 'function') {
      plugin.install.apply(plugin, args)
    } else if (typeof plugin === 'function') {
      plugin.apply(null, args)
    }
    // 添加到已安装插件
    installedPlugins.push(plugin)
    return this
  }
}
```

### initMixin
`Vue.mixin`用于合并实例对象的options，使用了`util`文件中定义的`mergeOptions`方法
```javascript
export function initMixin (Vue: GlobalAPI) {
  Vue.mixin = function (mixin: Object) {
    this.options = mergeOptions(this.options, mixin)
    return this
  }
}
```

### initExtend
`initExtend`方法在Vue上添加了`Vue.cid`静态属性，和`Vue.extend`静态方法。
```javascript
export function initExtend (Vue: GlobalAPI) {
  /*
    每个构造函数实例（包括Vue本身）都会有一个唯一的cid
    它为我们能够创造继承创建自构造函数并进行缓存创造了可能
  */
  Vue.cid = 0
  let cid = 1

  /*
    使用基础 Vue 构造器，创建一个“子类”。
    其实就是扩展了基础构造器，形成了一个可复用的有指定选项功能的子构造器。
    参数是一个包含组件option的对象。
  */
  Vue.extend = function (extendOptions: Object): Function {
    ……
  }
}

```

### initAssetRegisters
也是比较好理解的，通过遍历添加了`Vue.component`，`Vue.directive`，`Vue.filter`三个静态方法
```javasript
export function initAssetRegisters (Vue: GlobalAPI) {
  /**
   * Create asset registration methods.
   */
  ASSET_TYPES.forEach(type => {
    Vue[type] = function (
      id: string,
      definition: Function | Object
    ): Function | Object | void {
      if (!definition) {
        return this.options[type + 's'][id]
      } else {
        /* istanbul ignore if */
        if (process.env.NODE_ENV !== 'production' && type === 'component') {
          validateComponentName(id)
        }
        if (type === 'component' && isPlainObject(definition)) {
          definition.name = definition.name || id
          definition = this.options._base.extend(definition)
        }
        if (type === 'directive' && typeof definition === 'function') {
          definition = { bind: definition, update: definition }
        }
        this.options[type + 's'][id] = definition
        return definition
      }
    }
  })
}
```
`ASSET_TYPES`常量是从`shared/constants.js`引入的，打开后我们可以看到
```
export const ASSET_TYPES = [
  'component',
  'directive',
  'filter'
]
```

### 全局API整理
最后对 Vue 构造函数全局API(静态属性和方法)进行一下整理，便于看源码时查看方法的对应位置。
```javascript
// initGlobalAPI  global-api/index.js
Vue.config
Vue.util = {
  warn,
  extend,
  mergeOptions,
  defineReactive
}
Vue.set = set
Vue.delete = del
Vue.nextTick = nextTick
Vue.options = {
  components: {
    KeepAlive
  },
  directives: Object.create(null),
  filters: Object.create(null),
  _base: Vue
}

// initUse global-api/use.js
Vue.use = function (plugin: Function | Object) {}

// initMixin  global-api/mixin.js
Vue.mixin = function (mixin: Object) {}

// initExtend  global-api/extend.js
Vue.cid = 0
Vue.extend = function (extendOptions: Object): Function {}

// initAssetRegisters  global-api/assets.js
Vue.component =
Vue.directive =
Vue.filter = function (
  id: string,
  definition: Function | Object
): Function | Object | void {}

// entry-runtime-with-compiler.js
Vue.compile = compileToFunctions
```