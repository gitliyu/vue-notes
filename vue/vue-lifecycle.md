# 生命周期

生命周期是Vue中比较重要的一个概念，在Vue的执行过程中会在不同的时期创建钩子函数，方便我们在开发中使用

这一篇就从源码角度来分析一下生命周期的整个流程，首先贴一下官网的生命周期流程图
!['lifecycle'](https://github.com/gitliyu/vue-notes/blob/master/images/lifecycle.png)

### beforeCreate & created
通过看图可以发现，这两个钩子是在Vue初始化阶段开始挂载的，关于初始化流程在最早的一篇['Vue实例化过程'](https://github.com/gitliyu/vue-notes/blob/master/notes/vue-define.md)有这部分的介绍，我们直接来到`initMixin`方法，找到一下代码，位于`src/core/instance/init.js`
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
可以看到`beforeCreate`和`created`的钩子是在`initState`前后被触发的，`initState`的作用是初始化`props、methods、data、computed与watch`等属性，所以`beforeCreate`的钩子函数中就不能获取到`props、data`中定义的值，也不能调用`methods`中定义的函数

先来介绍下挂载生命周期钩子函数的方法`callHook`，位于`src/core/instance/lifecycle.js`
```javascript
export function callHook (vm: Component, hook: string) {
  pushTarget()
  // 拿到$options中定义的生命周期函数
  const handlers = vm.$options[hook]
  if (handlers) {
    for (let i = 0, j = handlers.length; i < j; i++) {
      try {
        handlers[i].call(vm)
      } catch (e) {
        handleError(e, vm, `${hook} hook`)
      }
    }
  }
  if (vm._hasHookEvent) {
    vm.$emit('hook:' + hook)
  }
  popTarget()
}
```
`callHook`方法接受两个参数，vm以及生命周期函数名

首尾执行的`pushTarget`和`popTarget`方法是为了在调用生命周期钩子时禁用dep依赖收集，避免在某些生命周期钩子中收集冗余的依赖

可以看到`callHook`的主要功能就是拿到`$options`上面定义的生命周期函数集合，判断存在时，就会进行调用

在看这段代码时，还发现了一个彩蛋，就是最后的这部分代码，`vm._hasHookEvent`是在`initEvents` 函数中定义的，它的作用是判断是否存在生命周期钩子的事件侦听器，初始化值为false，当组件检测到存在生命周期钩子的事件侦听器时，会将`vm._hasHookEvent`设置为true，在每次生命周期函数调用时，都会使用`$emit`发送一个事件，那么就发现了一个很有意思的写法，举个栗子
```
<child @hook:created="handleChildCreated">	
```
这样我们就可以在监听组件`created`方法的触发，其他生命周期方法同理，这个操作官网都没有写，可以算是意外收获吧

### beforeMount & mounted
回到`initMixin`方法，我们可以在底部看到以下代码
```javascript
if (vm.$options.el) {
  vm.$mount(vm.$options.el)
}
```
和生命周期的流程图是对应的，如果`$options`中定义了`el`，就调用原型上的`$mount`方法进行挂载，找到`$mount`的定义，位于`src/platforms/web/runtime/index.js`
```javascript
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  el = el && inBrowser ? query(el) : undefined
  return mountComponent(this, el, hydrating)
}
```
实际调用了`mountComponent`，只贴一下主要部分代码吧，位于`src/core/instance/lifecycle.js`
```javascript
export function mountComponent (
  vm: Component,
  el: ?Element,
  hydrating?: boolean
): Component {
  vm.$el = el
  if (!vm.$options.render) {
  	// 创建空节点
    vm.$options.render = createEmptyVNode
    ……
  }
  // 调beforeMount钩子函数并且触发beforeMount钩子事件
  callHook(vm, 'beforeMount')

  // 初次执行的_update方法，之前有过介绍
  let updateComponent
  if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
    updateComponent = () => {
      const name = vm._name
      const id = vm._uid
      const startTag = `vue-perf-start:${id}`
      const endTag = `vue-perf-end:${id}`

      mark(startTag)
      const vnode = vm._render()
      mark(endTag)
      measure(`vue ${name} render`, startTag, endTag)

      mark(startTag)
      vm._update(vnode, hydrating)
      mark(endTag)
      measure(`vue ${name} patch`, startTag, endTag)
    }
  } else {
    updateComponent = () => {
      vm._update(vm._render(), hydrating)
    }
  }

  // 实例化Watcher
  new Watcher(vm, updateComponent, noop, {
    before () {
      if (vm._isMounted) {
        callHook(vm, 'beforeUpdate')
      }
    }
  }, true /* isRenderWatcher */)
  hydrating = false

  // 渲染完成
  if (vm.$vnode == null) {
    vm._isMounted = true
    callHook(vm, 'mounted')
  }
  return vm
}
```
这部分代码在['数据驱动视图的方法'](https://github.com/gitliyu/vue-notes/blob/master/notes/vue-render.md)有过介绍，在创建空节点之后触发`beforeMount`钩子，之后执行首次渲染的`_update`方法和`_render`方法来创建VNode和真实dom，执行完毕后触发`mounted`钩子

### beforeUpdate & updated
`beforeUpdate`方法上面已经出现了，在`mountComponent`中初始化`Watcher`时就会被触发，接下来找一下`updated`，
首先看一下`Watcher`，位于`sr/core/observer/watcher.js`

具体之前也有过介绍['Observer与响应式数据'](https://github.com/gitliyu/vue-notes/blob/master/notes/vue-observer.md)，找到数据更新的方法`update`
```javascript
update () {
  if (this.computed) {
    if (this.dep.subs.length === 0) {
      this.dirty = true
    } else {
      this.getAndInvoke(() => {
        this.dep.notify()
      })
    }
  } else if (this.sync) {
    this.run()
  } else {
    queueWatcher(this)
  }
}
```
异步更新时触发`queueWatcher`方法加入异步更新队列，位于`src/core/observer/scheduler.js`，然后按步骤找
```javascript
function flushSchedulerQueue () {
  ……
  // call component updated and activated hooks
  callActivatedHooks(activatedQueue)
  callUpdatedHooks(updatedQueue)
  ……
}
```
同文件中的`callUpdatedHooks`
```javascript
function callUpdatedHooks (queue) {
  let i = queue.length
  while (i--) {
    const watcher = queue[i]
    const vm = watcher.vm
    if (vm._watcher === watcher && vm._isMounted) {
      callHook(vm, 'updated')
    }
  }
}
```
`callUpdatedHooks`方法对`Watcher`队列做遍历，只有满足当前`Watcher`为`vm._watcher`以及组件已经`mounted`这两个条件，才会触发` updated`钩子函数

### beforeDestroy & destroyed
这两个钩子是在组件销毁时才会触发的，也就是调用了销毁方法`vm.$destroy`，回到`src/core/instance/lifecycle`
```javascript
Vue.prototype.$destroy = function () {
  const vm: Component = this
  // 如果已经被销毁，返回
  if (vm._isBeingDestroyed) {
    return
  }
  // 调用beforeDestroy钩子
  callHook(vm, 'beforeDestroy')
  // 标记
  vm._isBeingDestroyed = true
  // 将自身从父节点移除
  const parent = vm.$parent
  if (parent && !parent._isBeingDestroyed && !vm.$options.abstract) {
    remove(parent.$children, vm)
  }
  // 将自身所有依赖释放
  if (vm._watcher) {
    vm._watcher.teardown()
  }
  let i = vm._watchers.length
  while (i--) {
    vm._watchers[i].teardown()
  }
  // 删除自身observer
  if (vm._data.__ob__) {
    vm._data.__ob__.vmCount--
  }
  vm._isDestroyed = true
  // 销毁VNode
  vm.__patch__(vm._vnode, null)
  // 调用destroyed钩子
  callHook(vm, 'destroyed')
  // 关闭所有事件监听
  vm.$off()
  // 清空自身__vue__
  if (vm.$el) {
    vm.$el.__vue__ = null
  }
  // 删除父节点
  if (vm.$vnode) {
    vm.$vnode.parent = null
  }
}
```
方法最初调用时触发`destroy`，然后在进行一系列销毁动作之后，触发`destroyed
