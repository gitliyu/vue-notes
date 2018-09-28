# Install安装方法
首先都知道的一点是，`vue-router`使用`Vue.use`安装使用
```javascript
import Router from 'vue-router'

Vue.use(Router)
```
关于`Vue.use`方法，来看一下官网的介绍
> 安装`Vue.js`插件。如果插件是一个对象，必须提供`install`方法。如果插件是一个函数，它会被作为`install`方法。`install` 方法调用时，会将`Vue`作为参数传入

找到`vue-router`的`install`方法，位于`src/install.js`
```javascript
export function install (Vue) {
  // 判断已经被安装，直接返回 
  if (install.installed && _Vue === Vue) return
  // 标识已安装
  install.installed = true

  // 记录当前vue实例
  _Vue = Vue

  // 方法，检查变量是否已定义
  const isDef = v => v !== undefined

  const registerInstance = (vm, callVal) => {
    let i = vm.$options._parentVnode
    if (isDef(i) && isDef(i = i.data) && isDef(i = i.registerRouteInstance)) {
      i(vm, callVal)
    }
  }

  // 在实例中混入钩子函数
  Vue.mixin({
    beforeCreate () {
      // 判断当前options中是否有router
      if (isDef(this.$options.router)) {
      	// 有的话，说明当前是根组件
      	// 将当前实例保存为_routerRoot
        this._routerRoot = this
        // 保存router
        this._router = this.$options.router
        // 调用router的init方法初始化路由
        this._router.init(this)
        // 将_route设置为响应式属性
        Vue.util.defineReactive(this, '_route', this._router.history.current)
      } else {
      	// 记录根节点
        this._routerRoot = (this.$parent && this.$parent._routerRoot) || this
      }
      registerInstance(this, this)
    },
    destroyed () {
      registerInstance(this)
    }
  })

  // 在原型上定义$router和$route的响应式属性
  Object.defineProperty(Vue.prototype, '$router', {
    get () { return this._routerRoot._router }
  })

  Object.defineProperty(Vue.prototype, '$route', {
    get () { return this._routerRoot._route }
  })

  // 注册为全局组件
  Vue.component('RouterView', View)
  Vue.component('RouterLink', Link)

  const strats = Vue.config.optionMergeStrategies
  // 对于路由钩子使用相同的钩子合并策略
  strats.beforeRouteEnter = strats.beforeRouteLeave = strats.beforeRouteUpdate = strats.created
}
```
总结一下`router`安装过程中做的几件事
- 对`Vue`实例混入生命周期钩子
- 在`Vue`原型上定义`$router`和`$route`，方便所有组件可以获取这两个属性） 
- 全局注册`router-link`和`router-view`两个组件