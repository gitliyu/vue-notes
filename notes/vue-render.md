# 数据驱动视图的方式

参考： ['vue2.0-source'](https://github.com/liutao/vue2.0-source)

我们上文提到过（['Observer与响应式数据'](https://github.com/gitliyu/vue-notes/blob/master/notes/vue-observer.md)），当页面绑定的数据修改时，会触发监听该数据的watcher对象更新，触发回调方法，驱动视图进行更新，这一篇主要介绍一下视图更新主要需要用到的两个方法：`_render`和`_update`

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