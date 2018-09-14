# 数据驱动视图的方法

这一篇介绍一下视图更新主要需要用到的两个方法：`_render`和`_update`

### Render
在['Vue实例化过程'](https://github.com/gitliyu/vue-notes/blob/master/notes/vue-define.md)中，曾经简单对Render方法进行了介绍，方法位于`src/core/instance/render.js`

```javascript
Vue.prototype._render = function (): VNode {
  const vm: Component = this
  // 从$options中获取render方法和父节点
  const { render, _parentVnode } = vm.$options

  // 对于插槽的重复性检查
  if (process.env.NODE_ENV !== 'production') {
    for (const key in vm.$slots) {
      // $flow-disable-line
      vm.$slots[key]._rendered = false
    }
  }

  // 作用域slot的继承
  if (_parentVnode) {
    vm.$scopedSlots = _parentVnode.data.scopedSlots || emptyObject
  }

  // 设置父节点
  vm.$vnode = _parentVnode
  // render self
  let vnode
  // vm._renderProxy方法下面介绍
  // 调用createElement方法创建VNode
  try {
    vnode = render.call(vm._renderProxy, vm.$createElement)
  } catch (e) {
    handleError(e, vm, `render`)
    // return error render result,
    // or previous vnode to prevent render error causing blank component
    /* istanbul ignore else */
    if (process.env.NODE_ENV !== 'production') {
      if (vm.$options.renderError) {
        try {
          vnode = vm.$options.renderError.call(vm._renderProxy, vm.$createElement, e)
        } catch (e) {
          handleError(e, vm, `renderError`)
          vnode = vm._vnode
        }
      } else {
        vnode = vm._vnode
      }
    } else {
      vnode = vm._vnode
    }
  }
  // 如果VNode节点没有创建成功则创建一个空节点
  if (!(vnode instanceof VNode)) {
    if (process.env.NODE_ENV !== 'production' && Array.isArray(vnode)) {
      warn(
        'Multiple root nodes returned from render function. Render function ' +
        'should return a single root node.',
        vm
      )
    }
    vnode = createEmptyVNode()
  }
  // 设置父节点
  vnode.parent = _parentVnode
  return vnode
}
```
其中用到的`_renderProxy`方法是在实例化过程中被挂载在对象属性上的
```javascript
initProxy = function initProxy (vm) {
  if (hasProxy) {
    // determine which proxy handler to use
    const options = vm.$options
    const handlers = options.render && options.render._withStripped
      ? getHandler
      : hasHandler
    vm._renderProxy = new Proxy(vm, handlers)
  } else {
    vm._renderProxy = vm
  }
}
```
可以看到通过`new Proxy`的方式中对vm对象进行劫持，也就是使用handlers进行处理，这里的handlers对应的是hasHandler
```javascript
const hasHandler = {
  has (target, key) {
    const has = key in target
    const isAllowed = allowedGlobals(key) || (typeof key === 'string' && key.charAt(0) === '_')
    if (!has && !isAllowed) {
      warnNonPresent(target, key)
    }
    return has || !isAllowed
  }
}
```
首先判断实例中已经定义该属性，然后再判断属性的格式是否与api名字重复以及是否是以下划线命名

总结一下，`_render`函数主要是获取到vm.$options.render函数，通过使用$createElement函数，创建并返回一个VNode，同时经过proxy代理检查属性是否满足要求

### Update
我之前介绍过（['Observer与响应式数据'](https://github.com/gitliyu/vue-notes/blob/master/notes/vue-observer.md)），当页面绑定的数据修改时，会触发监听该数据的Watcher对象更新，触发回调方法，驱动视图进行更新
而Watcher是在mount的时候被定义的，我们来到相关文件`src/core/instance/lifecycle.js`
```javascript
new Watcher(vm, updateComponent, noop, {
  before () {
    if (vm._isMounted) {
      callHook(vm, 'beforeUpdate')
    }
  }
}, true /* isRenderWatcher */)
```
可以看到Watcher传入的回调方法是`updateComponent`
```javascript
let updateComponent
/* istanbul ignore if */
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
```
`updateComponent`实际上是调用了`_render`方法创建VNode，之后调用`_update`方法进行更新，`_update`方法也是在这个文件定义的
```javascript
Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {
  const vm: Component = this
  // 保存原属性
  const prevEl = vm.$el
  const prevVnode = vm._vnode
  const prevActiveInstance = activeInstance
  activeInstance = vm
  // 更新_vnode
  vm._vnode = vnode
  if (!prevVnode) {
    // 判断之前没有vnode，初次渲染
    vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */)
  } else {
    // 更新
    vm.$el = vm.__patch__(prevVnode, vnode)
  }
  activeInstance = prevActiveInstance
  // 更新新的实例对象的__vue__
  if (prevEl) {
    prevEl.__vue__ = null
  }
  if (vm.$el) {
    vm.$el.__vue__ = vm
  }
  // if parent is an HOC, update its $el as well
  if (vm.$vnode && vm.$parent && vm.$vnode === vm.$parent._vnode) {
    vm.$parent.$el = vm.$el
  }
}
```
主要是执行了`__patch__`渲染函数，传入原生dom节点和生成的VNode，原型上的`__patch__`方法是在加载框架的时候注入的

src/platforms/web/runtime/index.js
```javascript
Vue.prototype.__patch__ = inBrowser ? patch : noop
```
src/platforms/web/runtime/patch.js
```javascript
import { createPatchFunction } from 'core/vdom/patch'

export const patch: Function = createPatchFunction({ nodeOps, modules })
```
这里，nodeOps为封装的dom操作方法，modules为属性、指令等相关方法，`createPatchFunction`方法涉及到VirtualDOM的比较算法，会在下一篇进行介绍