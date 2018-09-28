# VueRouter类介绍
`VueRouter`是我们在`import`引入时拿到的一个`es6`方式定义的`class`，位于`src/index.js`，是`vue-router`的主要内容

由于内容比较多，所以分开对每个部分进行介绍，首先看一下`constructor`的内容
```javascript
constructor (options: RouterOptions = {}) {
  // 用记录当前vue实例
  this.app = null
  this.apps = []
  // 路由配置项
  this.options = options
  // 路由守卫
  this.beforeHooks = []
  this.resolveHooks = []
  this.afterHooks = []
  // 对routes进行匹配处理
  this.matcher = createMatcher(options.routes || [], this)

  // 默认使用hash模式
  let mode = options.mode || 'hash'
  // 如果设置为history模式，pushState不可用并且设置了fallback，切换为hash模式
  this.fallback = mode === 'history' && !supportsPushState && options.fallback !== false
  if (this.fallback) {
    mode = 'hash'
  }
  // 非浏览器环境（node）使用abstract模式
  if (!inBrowser) {
    mode = 'abstract'
  }
  this.mode = mode

  // 根据mode模式对路径跳转方式做不同的封装，如果不是三种模式之一，报错
  switch (mode) {
    case 'history':
      this.history = new HTML5History(this, options.base)
      break
    case 'hash':
      this.history = new HashHistory(this, options.base, this.fallback)
      break
    case 'abstract':
      this.history = new AbstractHistory(this, options.base)
      break
    default:
      if (process.env.NODE_ENV !== 'production') {
        assert(false, `invalid mode: ${mode}`)
      }
  }
}
```
总结一下`constructor`做的几件事
- 初始化定义变量
- 调用`createMatcher`方法对`route`进行匹配处理
- 设置`mode`模式，并根据不同模式对`history`进行封装定义

> 关于`createMatcher`和`HTML5History`这些在之后进行介绍

```javascript
init (app: any /* Vue component instance */) {
  // 判断未被安装，报错
  process.env.NODE_ENV !== 'production' && assert(
    install.installed,
    `not installed. Make sure to call \`Vue.use(VueRouter)\` ` +
    `before creating root instance.`
  )

  this.apps.push(app)

  // 如果根组件路由已经被初始化
  if (this.app) {
    return
  }

  this.app = app

  const history = this.history

  // 初始化时，根据history类型对初始路径进行跳转
  if (history instanceof HTML5History) {
    history.transitionTo(history.getCurrentLocation())
  } else if (history instanceof HashHistory) {
    const setupHashListener = () => {
      history.setupListeners()
    }
    history.transitionTo(
      history.getCurrentLocation(),
      setupHashListener,
      setupHashListener
    )
  }

  history.listen(route => {
    this.apps.forEach((app) => {
      app._route = route
    })
  })
}
```
`init`方法完成了对路由的初始化设置，在`install`方法中混入`beforeCreate`钩子函数时被调用

```javascript
beforeEach (fn: Function): Function {
  return registerHook(this.beforeHooks, fn)
}

beforeResolve (fn: Function): Function {
  return registerHook(this.resolveHooks, fn)
}

afterEach (fn: Function): Function {
  return registerHook(this.afterHooks, fn)
}
```
接下来是对于导航守卫的定义，调用了`registerHook`方法进行了定义
```javascript
function registerHook (list: Array<any>, fn: Function): Function {
  list.push(fn)
  return () => {
    const i = list.indexOf(fn)
    if (i > -1) list.splice(i, 1)
  }
}
```
what?这怎么这么简单，这里只是将回调函数方法`push`进对应的数组中，并且进行了去重的操作

> 未完待续