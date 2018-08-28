# Global API
上一篇在入口文件中看到了`initGlobalAPI`方法，那么就来看一下`global-api`这个目录吧，Vue的静态方法大多都是在该文件夹定义的

参考： ['learnVue'](https://github.com/answershuto/learnVue)
### 文件结构
首先看一下文件结构，`index.js`里面的引入路径可以很清楚的看出来各个文件的作用
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
> 未完待续