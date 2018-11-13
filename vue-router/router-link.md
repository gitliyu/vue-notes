# RouterLink组件
先来介绍一下`vue-router`定义的`router-link`组件，如果对组件使用不够了解的，可以查看['官方文档'](https://router.vuejs.org/zh/api/#router-link)，源码位于`/src/components/link.js`
```javascript
const toTypes: Array<Function> = [String, Object]
const eventTypes: Array<Function> = [String, Array]

export default {
  name: 'RouterLink',
  props: {
    // 表示目标路由，可以是字符串或者对象
    to: {
      type: toTypes,
      required: true
    },
    // 替换标签，默认是a标签
    tag: {
      type: String,
      default: 'a'
    },
    // 判断路由激活时，是否用全匹配
    // 只有绝对相等的路由才会增加 activeClass，默认是包含关系
    exact: Boolean,
    // 在当前（相对）路径附加路径
    append: Boolean,
    // 使用replace方法做路由替换
    replace: Boolean,
    // 链接匹配时的类名
    activeClass: String,
    // 当链接被精确匹配的时候应该激活的 class
    exactActiveClass: String,
    // 声明可以用来触发导航的事件，可以是字符串或数组
    event: {
      type: eventTypes,
      default: 'click'
    }
  },
  render (h: Function) {
    // router实例和当前激活的route对象
    const router = this.$router
    const current = this.$route
    // 根据当前目标和当前激活的route匹配结果
    const { location, route, href } = router.resolve(this.to, current, this.append)

    const classes = {}
    // 获取全局的class
    const globalActiveClass = router.options.linkActiveClass
    const globalExactActiveClass = router.options.linkExactActiveClass
    // 使用配置的全局class或者'router-link-active'
    const activeClassFallback = globalActiveClass == null
      ? 'router-link-active'
      : globalActiveClass
    const exactActiveClassFallback = globalExactActiveClass == null
      ? 'router-link-exact-active'
      : globalExactActiveClass
    // 优先使用组件上定义的class
    const activeClass = this.activeClass == null
      ? activeClassFallback
      : this.activeClass
    const exactActiveClass = this.exactActiveClass == null
      ? exactActiveClassFallback
      : this.exactActiveClass
    // 与当前路由相比较的路由，考虑到命名路由，不一定有path
    const compareTarget = location.path
      ? createRoute(null, location, null, router)
      : route

    // 严格模式的class，判断是否是相同路由（path query params hash）
    classes[exactActiveClass] = isSameRoute(current, compareTarget)
    // 对于activeClass，如果是严格模式的话，使用严格模式的class
    // 否则就走包含逻辑（path包含，query包含 hash为空或者相同） 
    classes[activeClass] = this.exact
      ? classes[exactActiveClass]
      : isIncludedRoute(current, compareTarget)

    const handler = e => {
      // guardEvent判断是否点击事件是否允许
      if (guardEvent(e)) {
      	// 根据组件上的replace属性，判断是replace还是push逻辑
        if (this.replace) {
          router.replace(location)
        } else {
          router.push(location)
        }
      }
    }

    // 点击事件
    const on = { click: guardEvent }
    // 根据声明的事件类型，如果是数组，将数组遍历绑定事件
    // 如果是函数，直接绑定事件
    if (Array.isArray(this.event)) {
      this.event.forEach(e => { on[e] = handler })
    } else {
      on[this.event] = handler
    }

    // 创建元素时需要的数据
    const data: any = {
      class: classes
    }

    if (this.tag === 'a') {
      // 修改a标签的href属性
      data.on = on
      data.attrs = { href }
    } else {
      // 递归所有子级，查找第一个a标签，修改href属性
      const a = findAnchor(this.$slots.default)
      if (a) {
        a.isStatic = false
        const aData = a.data = extend({}, a.data)
        aData.on = on
        const aAttrs = a.data.attrs = extend({}, a.data.attrs)
        aAttrs.href = href
      } else {
        // 没有a标签的话就给自身绑定事件
        data.on = on
      }
    }

    // 渲染组件
    return h(this.tag, data, this.$slots.default)
  }
}
```
总结一下`router-link`组件的功能，就是在其点击的时候根据设置的`to`的值去调用`router`的`push`或者`replace`方法 来更新路由的，同时会检查自身路由匹配来附加`activeClass`

再来看一下上面用到的两个方法
```javascript
// 判断是否点击事件是否允许
function guardEvent (e) {
  // 带有功能键的点击
  if (e.metaKey || e.altKey || e.ctrlKey || e.shiftKey) return
  // 已阻止的
  if (e.defaultPrevented) return
  // 右击事件
  if (e.button !== undefined && e.button !== 0) return
  // target="_blank
  if (e.currentTarget && e.currentTarget.getAttribute) {
    const target = e.currentTarget.getAttribute('target')
    if (/\b_blank\b/i.test(target)) return
  }
  // 阻止默认行为
  if (e.preventDefault) {
    e.preventDefault()
  }
  return true
}

// 查找a标签
function findAnchor (children) {
  if (children) {
    let child
    // 遍历所有子节点
    for (let i = 0; i < children.length; i++) {
      child = children[i]
      // 找到a标签就返回，否则继续遍历子节点
      if (child.tag === 'a') {
        return child
      }
      if (child.children && (child = findAnchor(child.children))) {
        return child
      }
    }
  }
}
```
