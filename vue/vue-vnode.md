# Virtual DOM 介绍

众所周知浏览器对于DOM操作的成本是很大的，对于复杂交互而言，需要不断的重绘DOM结构，从而对于浏览器的性能造成了很大的压力

### 为什么要用 Virtual DOM
用我们传统的开发模式操作DOM时，比如在一次操作中，我需要更新10个DOM节点，浏览器收到第一个DOM请求后会马上执行流程，紧接着下一个DOM更新请求，最终执行10次，每次都需要执行后DOM节点都会发生变化，下次更新需要重新查询，举个例子
```javascript
<body>
  <ul>
    <li>item1</li>  
    <li>item2</li>  
    <li>item3</li>  
    <li>item4</li>
    <li>item5</li>
  </ul>
</body>
<script type="text/javascript">
  let ul = $('ul');
  ul.children('li').eq(0).remove();
  ul.children('li').eq(0).remove();
  ul.children('li').eq(0).remove();
  ul.children('li').eq(0).remove();
</script>
```
上面的几次操作，每次将第一个li标签删除，而每次删除导致了DOM结构的变化，下一次操作时会重新查询并执行删除，白白浪费性能

而`Virtual DOM`的主要思想就是将DOM抽象成一个以对象为节点的虚拟DOM树，以`VNode`节点模拟真实DOM，作为真实DOM的一层抽象，在由于交互等因素需要视图更新时，先通过对节点数据进行`diff`后得到差异结果后，再一次性对DOM进行批量更新操作，所有复杂曲折的更新逻辑都由虚拟的`Virtual DOM`处理完成，只将最终的更新结果发送给浏览器中的DOM树执行，这样就避免了冗余琐碎的DOM树操作负担，进而有效提高了性能

### VNode
先来看一下`VNode`节点的结构，每一个`VNode`对应一个真实的DOM对象，代码位于`src/core/vdom/vnode.js`
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
    // 节点对应的对象，VNodeData类型，包含了标签上的属性信息等
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

下面举个例子来看一下`VNode`的结构，如下的DOM结构
```html
<div id="app">
  <div>Hello</div>
  <ul>
    <li class="item">item 1</li>
    <li class="item">item 2</li>
  </ul>
</div>
```
来简单的写一下对应的`VNode`对象
```javascript
{
  "tag": "div",
  "data": {
    "attr": {"id": "app"}
  },
  "children": [
    {
      "tag": "div",
      "text": "Hello"
    },
    {
      "tag": "ul",
      "children": [
        {
          "tag": "li",
          "data": {
            "staticClass": "item"
          },
          "text": "item 1"
        },
        {
          "tag": "li",
          "data": {
            "staticClass": "item"
          },
          "text": "item 2"
        }
      ]
    }
  ]
}
```

### createElement
Vue使用`createElement`方法创建`VNode`对象，方法在`src/core/vdom/create-elemenet.js`中
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
> 关于`Virtual DOM`的相关算法和渲染会在下篇详细介绍