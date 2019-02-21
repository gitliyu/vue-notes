# 前端路由实现：Hash和History模式

前端路由需要实现的核心功能就是更新视图但不重新请求页面，目前在浏览器环境中这一功能的实现主要有两种方式：
1. URL中的`hash`("#")
2. HTML5中新增的`History API`

首先来回顾一下`VueRouter`在实例化时对于路由模式的定义规则，具体可以查看['VueRouter类介绍'](https://github.com/gitliyu/vue-notes/blob/master/vue-router/router-define.md)
1. 作为参数传入的字符串属性`mode`，对应着不同的`history`的实现类，两者对应关系如下：
	- history -> HTML5History
	- hash -> HashHistory
	- abstract -> AbstractHistory
2. 在初始化对应的`history`之前，会对`mode`做一些校验：默认为'hash'；若浏览器不支持`HTML5History`（通过supportsPushState变量判断），则`mode`强制设为'hash'；若不是在浏览器环境下运行，则`mode`强制设为'abstract'
3. `VueRouter`类中的`push()`, `replace()`等方法只是一个代理，实际是调用的具体`history`对象的对应方法，在`init()`方法中初始化时，也是根据`history`对象具体的类别执行不同操作

### HashHistory
首先来介绍一下浏览器`hash`的，比如
```
https://liyu.fun/#/index
```
`#`符号本身以及它后面的字符称之为`hash`，可通过`window.location.hash`属性读取，以上地址的`hash`值是`#/index`，它具有如下特点：
- `hash`虽然出现在URL中，但不会被包括在HTTP请求中，因此，当`hash`发生改变时不会重新加载页面
- 可以为hash的改变添加监听事件：
```javascript
window.addEventListener("hashchange", callback, false);
```
- 每一次改变`hash`，都会在浏览器的访问历史中增加一个记录

利用`hash`的以上特点，就可以来实现前端路由“更新视图但不重新请求页面”的功能了，接下来看一下代码，以`push()`为例

```javascript
push (location: RawLocation, onComplete?: Function, onAbort?: Function) {
  const { current: fromRoute } = this
  this.transitionTo(location, route => {
    pushHash(route.fullPath)
    handleScroll(this.router, route, fromRoute, false)
    onComplete && onComplete(route)
  }, onAbort)
}
```
`transitionTo`方法是父类中定义的是用来处理路由变化中的基础逻辑的，主要是依赖`pushHash`方法进行路由的切换
```javascript
function pushHash (path) {
  if (supportsPushState) {
    pushState(getUrl(path))
  } else {
    window.location.hash = path
  }
}
```
首先判断是否支持`History API`，如果支持的话，使用封装的`pushState`进行切换，否则直接修改浏览器`hash`，`hash`的改变会自动添加到浏览器的访问历史记录中。
接下来是视图的更新，我们来看父类中相关的一些代码：
```javascript
transitionTo (location: RawLocation, onComplete?: Function, onAbort?: Function) {
  const route = this.router.match(location, this.current)
  this.confirmTransition(route, () => {
    this.updateRoute(route)
    ……
  });
}

updateRoute (route: Route) {
  const prev = this.current
  this.current = route
  this.cb && this.cb(route)
  this.router.afterHooks.forEach(hook => {
    hook && hook(route, prev)
  })
}

listen (cb: Function) {
  this.cb = cb
}
```
可以看到，更新路由时调用了`History`中的`this.cb`方法，方法是通过History.listen(cb)进行设置的。回到`VueRouter`类定义中，找到了在`init`方法中对其进行了设置：
```javascript
init (app: any /* Vue component instance */) {
  this.apps.push(app);
  history.listen(route => {
    this.apps.forEach((app) => {
      app._route = route
    })
  })
}
```
接下来回到`install`方法，可以看到在安装时对实例进行了混入
```javascript
Vue.mixin({
  beforeCreate () {
    if (isDef(this.$options.router)) {
      this._routerRoot = this
      this._router = this.$options.router
      this._router.init(this)
      Vue.util.defineReactive(this, '_route', this._router.history.current)
    } else {
      this._routerRoot = (this.$parent && this.$parent._routerRoot) || this
    }
    registerInstance(this, this)
  }
})
```
可以看到这里在`beforeCreate`钩子中通过`Vue.util.defineReactive`定义了响应式的`_route`属性。所谓响应式属性，即当`_route`值改变时，会自动调用实例的`render`方法，来更新视图

总结一下从设置路由到更新视图的过程：
```javascript
$router.push() => HashHistory.push() => History.transitionTo() => History.updateRoute() => vm.render()
```

### HTML5 History
`History interface`是浏览器历史记录栈提供的接口，通过`back()`, `forward()`, `go()`等方法，我们可以读取浏览器历史记录栈的信息，进行各种跳转操作，具体可以查看文档['History API'](https://developer.mozilla.org/zh-CN/docs/Web/API/History_API)

从HTML5开始，`History interface`提供了两个新的方法：`pushState()`, `replaceState()`使得我们可以对浏览器历史记录栈进行修改：
```javascript
window.history.pushState(stateObject, title, URL)
window.history.replaceState(stateObject, title, URL)
```
- stateObject: 当浏览器跳转到新的状态时，将触发popState事件，该事件将携带这个stateObject参数的副本
- title: 所添加记录的标题
- URL: 所添加记录的URL

这两个方法有个共同的特点：当调用他们修改浏览器历史记录栈后，虽然当前URL改变了，这就为单页应用前端路由“更新视图但不重新请求页面”提供了基础。
来看一下这部分的源码
```javascript
push (location: RawLocation, onComplete?: Function, onAbort?: Function) {
  const { current: fromRoute } = this
  this.transitionTo(location, route => {
    pushState(cleanPath(this.base + route.fullPath))
    handleScroll(this.router, route, fromRoute, false)
    onComplete && onComplete(route)
  }, onAbort)
}

replace (location: RawLocation, onComplete?: Function, onAbort?: Function) {
  const { current: fromRoute } = this
  this.transitionTo(location, route => {
    replaceState(cleanPath(this.base + route.fullPath))
    handleScroll(this.router, route, fromRoute, false)
    onComplete && onComplete(route)
  }, onAbort)
}

export function pushState (url?: string, replace?: boolean) {
  saveScrollPosition()
  // try...catch the pushState call to get around Safari
  // DOM Exception 18 where it limits to 100 pushState calls
  const history = window.history
  try {
    if (replace) {
      history.replaceState({ key: _key }, '', url)
    } else {
      _key = genKey()
      history.pushState({ key: _key }, '', url)
    }
  } catch (e) {
    window.location[replace ? 'replace' : 'assign'](url)
  }
}

export function replaceState (url?: string) {
  pushState(url, true)
}
```
代码逻辑与`hash`模式基本类似，只不过将对`window.location.hash`直接进行赋值操作改为了调用`history.pushState()`和`history.replaceState()`方法

### 对比两种模式
当然除了`hash`和`history`模式外还有用于服务器端的`abstract`模式，这里就不进行介绍了，下面说一下这两种模式的区别
- `hash`比较丑，而且对于原有参数的识别更丑，比如`www.baidu.com/sites?id=111`，在页面打开后会被识别成`www.baidu.com/sites?id#/index`
- `pushState`设置的新URL可以是与当前URL同源的任意URL；而`hash`只可修改#后面的部分
- `pushState`设置的新URL可以与当前URL一模一样，这样也会把记录添加到栈中；而`hash`设置的新值必须与原来不一样才会触发记录添加到栈中
- `pushState`通过`stateObject`可以添加任意类型的数据到记录中；而`hash`只可添加短字符串
- `pushState`修改后的URL在请求时会将完整的URL来发送给后台，请求源可能会造成服务器响应无效参数或其他原因，使用时需要后台配合；`hash`的修改不会影响服务器请求