# VNode介绍

参考： ['vue-analysis'](https://github.com/ustbhuangyi/vue-analysis)

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
    // 是否作为跟节点插入
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
可以看到其实是对`_createElement`方法的一层封装，增加了对于参数的处理，接下来看一下具体的实现

> 未完待续