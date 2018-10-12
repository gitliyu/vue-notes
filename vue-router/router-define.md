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
这里的操作非常简单，只是将回调函数方法`push`进对应的数组中，并且进行了去重的操作，之后会在路由导航的各个阶段触发其中的回调方法
```javascript
onReady (cb: Function, errorCb?: Function) {
  this.history.onReady(cb, errorCb)
}

onError (errorCb: Function) {
  this.history.onError(errorCb)
}

push (location: RawLocation, onComplete?: Function, onAbort?: Function) {
  this.history.push(location, onComplete, onAbort)
}

replace (location: RawLocation, onComplete?: Function, onAbort?: Function) {
  this.history.replace(location, onComplete, onAbort)
}

go (n: number) {
  this.history.go(n)
}

back () {
  this.go(-1)
}

forward () {
  this.go(1)
}
```
这些就是`router`上面挂载的各种事件了，在调用时会根据`history`类型做出响应，除此之外还有剩下的几个事件
```javascript
getMatchedComponents (to?: RawLocation | Route): Array<any> {
  const route: any = to
    ? to.matched
      ? to
      : this.resolve(to).route
    : this.currentRoute
  // 没有目标路由时返回空数组
  if (!route) {
    return []
  }
  // 根据route查询匹配信息
  return [].concat.apply([], route.matched.map(m => {
    return Object.keys(m.components).map(key => {
      return m.components[key]
    })
  }))
}
```
`getMatchedComponents`方法返回目标位置或是当前路由匹配的组件数组
```javascript
resolve (
  to: RawLocation,
  current?: Route,
  append?: boolean
): {
  location: Location,
  route: Route,
  href: string,
  // for backwards compat
  normalizedTo: Location,
  resolved: Route
} {
  // 对路由进行解析处理
  const location = normalizeLocation(
    to,
    current || this.history.current,
    append,
    this
  )
  // 匹配对应的route
  const route = this.match(location, current)
  // 获取完整路径
  const fullPath = route.redirectedFrom || route.fullPath
  const base = this.history.base
  const href = createHref(base, fullPath, this.mode)
  return {
    location,
    route,
    href,
    // for backwards compat
    normalizedTo: location,
    resolved: route
  }
}
```
首先使用`normalizeLocation`对参数进行解析处理，拿到`location`对象，接下来对`location`对象进行处理

> 这里补充一下，使用路由对象的`resolve`方法解析路由，可以得到`location`,`router`,`href`等目标路由的信息，举个栗子

```javascript
const { href } = this.$router.resolve({
  name: 'foo',
  query: {
    bar
  }
})
window.open(href, '_blank')
```
方法接受的参数和`router-link`的`to`绑定的属性相同，会根据参数解析出路由信息，实现例如新页面打开路由页的功能

```javascript
addRoutes (routes: Array<RouteConfig>) {
  // 向匹配表中添加路由信息
  this.matcher.addRoutes(routes)
  if (this.history.current !== START) {
    this.history.transitionTo(this.history.getCurrentLocation())
  }
}
```
调用了`matcher`上的`addRoutes`方法动态添加路由，方法在`createMatcher`中被挂载，主要是调用了`createRouteMap`将路由信息添加到路由匹配表中，接下来来到文件底部
```javascript
VueRouter.install = install
VueRouter.version = '__VERSION__'

if (inBrowser && window.Vue) {
  window.Vue.use(VueRouter)
}
```
将`intall`安装方法和版本号注册在`VueRouter`类上，最后为了兼容`script`标签引入的方法，自调用了`Vue.use`安装