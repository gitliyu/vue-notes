# Patch与Diff算法

上一篇['数据驱动视图的方法'](https://github.com/gitliyu/vue-notes/blob/master/notes/vue-render.md)介绍到了，vue在调用`_update`方法更新虚拟dom元素的时候，执行的是`__patch__`渲染函数，那么什么是`__patch__`呢，下面介绍下主要的功能以及原理
### patch
patch将新老VNode节点进行比对，然后将根据两者的比较结果进行最小单位地修改视图，而不是将整个视图根据新的VNode重绘。patch的核心在于diff算法，这套算法可以高效地比较Virtual DOM的变更，得出变化以修改视图

> 太长不看: patch方法只修改前后有差异的VNode

首先看一下`patch`方法的代码，位于`src/core/vdom/patch.js`
```javascript
return function patch (oldVnode, vnode, hydrating, removeOnly) {
  // 判断VNode不存在且之前按VNode存在，调用销毁钩子
  if (isUndef(vnode)) {
    if (isDef(oldVnode)) invokeDestroyHook(oldVnode)
    return
  }

  // 是否为新的VNode
  let isInitialPatch = false
  // VNode队列
  const insertedVnodeQueue = []

  if (isUndef(oldVnode)) {
    // 如果没有旧的VNode，创建新节点
    isInitialPatch = true
    createElm(vnode, insertedVnodeQueue)
  } else {
    // 判断是否为真实dom元素
    const isRealElement = isDef(oldVnode.nodeType)
    if (!isRealElement && sameVnode(oldVnode, vnode)) {
      // 不是真实dom，且新旧VNode是同一节点，执行patch修改已有节点
      patchVnode(oldVnode, vnode, insertedVnodeQueue, removeOnly)
    } else {
      if (isRealElement) {
        // 判断oldVnode为真实dom元素，确认是否为服务器渲染环境或者是否可以执行成功的合并到真实dom中
        if (oldVnode.nodeType === 1 && oldVnode.hasAttribute(SSR_ATTR)) {
          oldVnode.removeAttribute(SSR_ATTR)
          hydrating = true
        }
        // 判断为服务端渲染
        if (isTrue(hydrating)) {
          if (hydrate(oldVnode, vnode, insertedVnodeQueue)) {
            // 调用insert钩子
            invokeInsertHook(vnode, insertedVnodeQueue, true)
            return oldVnode
          } else if (process.env.NODE_ENV !== 'production') {
            warn(
              'The client-side rendered virtual DOM tree is not matching ' +
              'server-rendered content. This is likely caused by incorrect ' +
              'HTML markup, for example nesting block-level elements inside ' +
              '<p>, or missing <tbody>. Bailing hydration and performing ' +
              'full client-side render.'
            )
          }
        }
        // 不是服务器渲染或者合并到真实 DOM 失败，创建一个空节点替换原有节点
        oldVnode = emptyNodeAt(oldVnode)
      }

      // 记录真实dom元素
      const oldElm = oldVnode.elm
      const parentElm = nodeOps.parentNode(oldElm)

      // 创建新节点
      createElm(
        vnode,
        insertedVnodeQueue,
        oldElm._leaveCb ? null : parentElm,
        nodeOps.nextSibling(oldElm)
      )

      // 更新父级节点元素
      if (isDef(vnode.parent)) {
        let ancestor = vnode.parent
        const patchable = isPatchable(vnode)
        while (ancestor) {
          for (let i = 0; i < cbs.destroy.length; ++i) {
            cbs.destroy[i](ancestor)
          }
          ancestor.elm = vnode.elm
          if (patchable) {
            for (let i = 0; i < cbs.create.length; ++i) {
              cbs.create[i](emptyNode, ancestor)
            }
            const insert = ancestor.data.hook.insert
            if (insert.merged) {
              for (let i = 1; i < insert.fns.length; i++) {
                insert.fns[i]()
              }
            }
          } else {
            registerRef(ancestor)
          }
          ancestor = ancestor.parent
        }
      }

      // 销毁旧节点
      if (isDef(parentElm)) {
        removeVnodes(parentElm, [oldVnode], 0, 0)
      } else if (isDef(oldVnode.tag)) {
        invokeDestroyHook(oldVnode)
      }
    }
  }
  
  // 调用insert钩子
  invokeInsertHook(vnode, insertedVnodeQueue, isInitialPatch)
  return vnode.elm
}
```
总结一下patch的逻辑
1. vnode不存在但是oldVnode存在，调用`invokeDestroyHook`来进行销毁
2. vnode存在但是oldVnode不存在，调用`createElm`来创建新节点
3. 当vnode和oldVnode都存在时
    1. oldVnode和vnode是同一个节点，就调用`patchVnode`来进行patch
    2. 当vnode和oldVnode不是同一个节点时，如果oldVnode是真实dom节点且为服务器端渲染，需要用`hydrate`函数将虚拟dom和真实dom进行映射，然后将oldVnode设置为对应的虚拟dom，找到oldVnode.elm的父节点，根据vnode创建一个真实dom节点并插入到该父节点中oldVnode.elm的位置

抛开生命周期钩子函数先不谈，里面值得注意的方法有几个
- createElm: 用于创建真实dom元素，这一篇就不具体看了
- sameVnode: 判断是否为同一节点
- patchVnode: 对新旧节点进行比较后更新

接下来看一下`sameVnode`和`patchVnode`这两个方法

### sameVnode

```javascript
function sameVnode (a, b) {
  return (
    a.key === b.key && (
      (
        a.tag === b.tag &&
        a.isComment === b.isComment &&
        isDef(a.data) === isDef(b.data) &&
        sameInputType(a, b)
      ) || (
        isTrue(a.isAsyncPlaceholder) &&
        a.asyncFactory === b.asyncFactory &&
        isUndef(b.asyncFactory.error)
      )
    )
  )
}
```
从代码中很清楚的可以看到，判断为相同VNode的几个条件
- key（当前节点的标识符）相同
- tag（当前节点的标签名）相同
- isComment（是否为注释节点）相同
- data（当前节点对应的对象，包含了具体的一些数据信息，是一个VNodeData类型）都有定义
- 当标签是<input>的时候，type必须相同

或者判断VNode为异步更新

### patchVnode

```javascript
function patchVnode (oldVnode, vnode, insertedVnodeQueue, removeOnly) {
  // 两个VNode节点相同则直接返回
  if (oldVnode === vnode) {
    return
  }

  const elm = vnode.elm = oldVnode.elm

  // 异步占位
  if (isTrue(oldVnode.isAsyncPlaceholder)) {
    if (isDef(vnode.asyncFactory.resolved)) {
      hydrate(oldVnode.elm, vnode, insertedVnodeQueue)
    } else {
      vnode.isAsyncPlaceholder = true
    }
    return
  }

  // 如果新旧VNode都是静态的，同时它们的key相同（代表同一节点）
  // 并且新的VNode是clone或者是标记了once（标记v-once属性，只渲染一次)
  // 那么只需要替换elm以及componentInstance即可
  if (isTrue(vnode.isStatic) &&
    isTrue(oldVnode.isStatic) &&
    vnode.key === oldVnode.key &&
    (isTrue(vnode.isCloned) || isTrue(vnode.isOnce))
  ) {
    vnode.componentInstance = oldVnode.componentInstance
    return
  }

  let i
  const data = vnode.data
  if (isDef(data) && isDef(i = data.hook) && isDef(i = i.prepatch)) {
    i(oldVnode, vnode)
  }

  // 记录新老节点的children
  const oldCh = oldVnode.children
  const ch = vnode.children
  if (isDef(data) && isPatchable(vnode)) {
    // 调用update回调以及update钩子
    for (i = 0; i < cbs.update.length; ++i) cbs.update[i](oldVnode, vnode)
    if (isDef(i = data.hook) && isDef(i = i.update)) i(oldVnode, vnode)
  }
  // 当前节点没有text文本时
  if (isUndef(vnode.text)) {
    if (isDef(oldCh) && isDef(ch)) {
      // 新旧VNode均有children子节点，则对子节点进行diff操作，调用updateChildren
      if (oldCh !== ch) updateChildren(elm, oldCh, ch, insertedVnodeQueue, removeOnly)
    } else if (isDef(ch)) {
      // 如果oldVnode没有子节点而vnode存在子节点，先清空elm的文本内容，然后为当前节点加入子节点
      if (isDef(oldVnode.text)) nodeOps.setTextContent(elm, '')
      addVnodes(elm, null, ch, 0, ch.length - 1, insertedVnodeQueue)
    } else if (isDef(oldCh)) {
      // 当vnode没有子节点而oldVnode有子节点的时候，则移除所有elm的子节点
      removeVnodes(elm, oldCh, 0, oldCh.length - 1)
    } else if (isDef(oldVnode.text)) {
      // 没有子节点时，清空elm的文本
      nodeOps.setTextContent(elm, '')
    }
  } else if (oldVnode.text !== vnode.text) {
    // 新旧VNodetext不一样时进行替换
    nodeOps.setTextContent(elm, vnode.text)
  }
  if (isDef(data)) {
    // 调用postpatch钩子
    if (isDef(i = data.hook) && isDef(i = i.postpatch)) i(oldVnode, vnode)
  }
}
```
总结一下`patchVnode`的逻辑
1. 如果新旧VNode相等，不需要`patch`，退出。 
2. 如果是异步占位，执行`hydrate`方法或者定义`isAsyncPlaceholder`为true，退出 
3. 如果新旧VNode都为静态，不用更新，将`oldVnode`的`componentInstance`实例传给当前`vnode`，退出
4. 执行`prepatch`钩子
5. 遍历调用`update`回调，并执行`update`钩子。 
6. 如果新旧VNode都有`children`，且`vnode`没有`text`、新旧VNode不相等，执行`updateChildren`方法更新子节点
7. 如果`vnode`有`children`，而`oldVnode`没有，清空文本并添加子节点
8. 如果`oldVnode`有`children`，而`vnode`没有，清空文并并移除子节点
9. 如果新旧VNode都没有`children`，`oldVnode`有`text`，`vnode`没有`text`，清空真实dom文本内容
10. 如果新旧VNode的`text`不同，更新真实dom元素文本内容
11. 调用 postpatch 钩子