# RouterView组件
`router-view`组件源码位于`/src/components/view.js`
```javascript
export default {
  name: 'RouterView',
  // 函数式组件，无状态，无实例，用一个简单的 render 函数返回虚拟节点来渲染自身
  functional: true,  
  props: {
  	// 视图名 默认为default
    name: {
      type: String,
      default: 'default'
    }
  },
  render (_, { props, children, parent, data }) {
    // 路由视图的标识符
    data.routerView = true

    // 直接使用父级的createElement()函数
    // 以便router-view呈现的组件能够解析指定的插槽
    const h = parent.$createElement
    const name = props.name
    const route = parent.$route
    // 缓存
    const cache = parent._routerViewCache || (parent._routerViewCache = {})

    // 确定当前路由的深度
    let depth = 0
    // keep-alive的标记
    let inactive = false
    while (parent && parent._routerRoot !== parent) {
      if (parent.$vnode && parent.$vnode.data.routerView) {
        depth++
      }
      if (parent._inactive) {
        inactive = true
      }
      parent = parent.$parent
    }
    data.routerViewDepth = depth

    // 如果组件是keep-alive状态，从缓存中得到需要渲染的组件(cache[name])，渲染组件
    if (inactive) {
      return h(cache[name], data, children)
    }

    // 得到相匹配的当前组件层级的路由记录
    const matched = route.matched[depth]
    // 如果没有匹配的，清空缓存，并返回一个空的vnode节点
    if (!matched) {
      cache[name] = null
      return h()
    }

    // 获取需要渲染的组件，并添加到缓存中
    const component = cache[name] = matched.components[name]

    // 附加实例注册钩子
	// 这将在实例注入的生命周期钩子中调用，设置匹配的实例
    data.registerRouteInstance = (vm, val) => {
      const current = matched.instances[name]
      if (
        (val && current !== vm) ||
        (!val && current === vm)
      ) {
        matched.instances[name] = val
      }
    }

    // 同时在prepatch钩子中注册实例
	//以防相同的组件实例在不同的路由中重用
    ;(data.hook || (data.hook = {})).prepatch = (_, vnode) => {
      matched.instances[name] = vnode.componentInstance
    }

    // 处理props
    let propsToPass = data.props = resolveProps(route, matched.props && matched.props[name])
    if (propsToPass) {
      // clone to prevent mutation
      propsToPass = data.props = extend({}, propsToPass)
      // pass non-declared props as attrs
      const attrs = data.attrs = data.attrs || {}
      for (const key in propsToPass) {
        if (!component.props || !(key in component.props)) {
          attrs[key] = propsToPass[key]
          delete propsToPass[key]
        }
      }
    }

    // 渲染组件
    return h(component, data, children)
  }
}
```
`router-view`组件的作用就是获取到对应的组建实例并渲染视图，有区别的地方就在对`keep-alive`组件的处理，如果是`inactive`状态，会从缓存中获取到视图对应的组件，否则从路由匹配中查询