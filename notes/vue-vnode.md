# VNode介绍

参考： ['vue-analysis'](https://github.com/ustbhuangyi/vue-analysis)   ['learnVue'](https://github.com/answershuto/learnVue)

众所周知浏览器对于dom操作的成本是很大的，对于复杂交互而言，传统的Javascript需要不断的渲染dom结构，从而对于浏览器的性能造成了很大的压力。

Vue将DOM抽象成一个以对象为节点的虚拟DOM树，以VNode节点模拟真实DOM，作为真实DOM的一层抽象

在数据变化的过程中，我们只需要对这棵抽象树进行操作，在这个过程中真实DOM不会发生变化，Vue会在下一个时间循环中（nextTick），使用`diff`算法将VNode节点对比之后进行差异修改，得到需要修改的最小单位，再将这个单位的视图进行更新，减少了不必要的dom操作，提高了性能

### VNode
先来看一下VNode节点的结构，`src/core/vdom/vnode.js`
```javascript
export default class VNode {
  constructor (
    tag?: string,
    data?: VNodeData,
    children?: ?Array<VNode>,
    text?: string,
    elm?: Node,
    context?: Component,
    componentOptions?: VNodeComponentOptions,
    asyncFactory?: Function
  ) {
  	// 标签名
    this.tag = tag
    // 节点对应的对象，VNodeData类型，包含了具体的一些数据信息
    this.data = data
    // 子节点数组
    this.children = children
    // 当前节点的文本
    this.text = text
    // 当前虚拟节点对应的真实dom节点
    this.elm = elm
    // 命名空间
    this.ns = undefined
    // 编译作用域
    this.context = context
    // 函数化组件作用域
    this.functionalContext = undefined
    // 节点的key属性，被当作节点的标志，用以优化
    this.key = data && data.key
    // 组件的option选项
    this.componentOptions = componentOptions
    // 组件的实例
    this.componentInstance = undefined
    // 父节点
    this.parent = undefined
    // true 原生HTML，false textContent
    this.raw = false
    // 静态节点标志
    this.isStatic = false
    // 是否作为根节点插入
    this.isRootInsert = true
    // 是否为注释节点
    this.isComment = false
    // 是否为克隆节点
    this.isCloned = false
    // 是否有v-once指令
    this.isOnce = false
  }
}
```
VNode又分为以下几个类型
- TextVNode：文本节点
- ElementVNode：普通元素节点
- ComponentVNode：组件节点
- EmptyVNode：空节点，或者说是没有内容的注释节点
- CloneVNode：克隆节点，可以是以上任意类型节点

### createElement
Vue使用`createElement`方法创建VNode对象，方法在`src/core/vdom/create-elemenet.js`中
```javascript
export function createElement (
  context: Component,
  tag: any,
  data: any,
  children: any,
  normalizationType: any,
  alwaysNormalize: boolean
): VNode | Array<VNode> {
  if (Array.isArray(data) || isPrimitive(data)) {
    normalizationType = children
    children = data
    data = undefined
  }
  if (isTrue(alwaysNormalize)) {
    normalizationType = ALWAYS_NORMALIZE
  }
  return _createElement(context, tag, data, children, normalizationType)
}
```
可以看到其实是对`_createElement`方法的一层封装，增加了对于参数的处理，主要有以下参数
- context: 表示 VNode 的上下文环境, Component 类型 
- tag: 表示标签，它可以是一个字符串，也可以是一个 Component
- data: VNode节点对应的对象，VNodeData类型
- children 表示当前 VNode 的子节点，任意类型，接下来需要被规范为标准的 VNode 数组
- normalizationType 表示子节点规范的类型，类型不同规范的方法也就不一样，它主要是参考 render 函数是编译生成的还是用户手写的

接下来看一下创建VNode节点的具体代码
```javascript
export function _createElement (
  context: Component,
  tag?: string | Class<Component> | Function | Object,
  data?: VNodeData,
  children?: any,
  normalizationType?: number
): VNode | Array<VNode> {
  // 如果data的__ob__已经定义，已经是响应式对象，那么创建一个空节点
  if (isDef(data) && isDef((data: any).__ob__)) {
    process.env.NODE_ENV !== 'production' && warn(
      `Avoid using observed data object as vnode data: ${JSON.stringify(data)}\n` +
      'Always create fresh vnode data objects in each render!',
      context
    )
    return createEmptyVNode()
  }
  // 动态组件的is属性，tag将被渲染
  if (isDef(data) && isDef(data.is)) {
    tag = data.is
  }
  // 没有tag属性，创建空节点
  if (!tag) {
    // in case of component :is set to falsy value
    return createEmptyVNode()
  }
  // key类型的验证
  if (process.env.NODE_ENV !== 'production' &&
    isDef(data) && isDef(data.key) && !isPrimitive(data.key)
  ) {
    if (!__WEEX__ || !('@binding' in data.key)) {
      warn(
        'Avoid using non-primitive value as key, ' +
        'use string/number value instead.',
        context
      )
    }
  }
  // 默认作用域插槽
  if (Array.isArray(children) &&
    typeof children[0] === 'function'
  ) {
    data = data || {}
    data.scopedSlots = { default: children[0] }
    children.length = 0
  }
  if (normalizationType === ALWAYS_NORMALIZE) {
    children = normalizeChildren(children)
  } else if (normalizationType === SIMPLE_NORMALIZE) {
    children = simpleNormalizeChildren(children)
  }
  let vnode, ns
  if (typeof tag === 'string') {
    let Ctor
    // 获取上下文环境和tag的名字空间
    ns = (context.$vnode && context.$vnode.ns) || config.getTagNamespace(tag)
    // 判断是否是保留的标签
    if (config.isReservedTag(tag)) {
      // 如果是保留的标签则创建一个相应节点
      vnode = new VNode(
        config.parsePlatformTagName(tag), data, children,
        undefined, undefined, context
      )
    } else if (isDef(Ctor = resolveAsset(context.$options, 'components', tag))) {
      // 从vm实例的option的components中寻找该tag，存在则就是一个组件，创建相应节点，Ctor为组件的构造类
      vnode = createComponent(Ctor, data, context, children, tag)
    } else {
      // 未知的元素，在运行时检查，因为父组件可能在序列化子组件的时候分配一个名字空间
      vnode = new VNode(
        tag, data, children,
        undefined, undefined, context
      )
    }
  } else {
    // tag不是字符串的时候则是组件的构造类
    vnode = createComponent(tag, data, context, children)
  }
  if (Array.isArray(vnode)) {
    return vnode
  } else if (isDef(vnode)) {
    // 如果有名字空间，则递归所有子节点应用该名字空间
    if (isDef(ns)) applyNS(vnode, ns)
    if (isDef(data)) registerDeepBindings(data)
    return vnode
  } else {
    // 如果vnode没有成功创建则创建空节点
    return createEmptyVNode()
  }
}
```

> 以后再补充吧，这部分我也没搞太清楚