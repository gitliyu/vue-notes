# Patch与Diff算法

上一篇['数据驱动视图的方法'](https://github.com/gitliyu/vue-notes/blob/master/notes/vue-render.md)介绍到了，vue在调用`_update`方法更新虚拟dom元素的时候，执行的是`__patch__`渲染函数，那么什么是`__patch__`呢，下面介绍下主要的功能以及原理
### patch
patch将新老VNode节点进行比对，然后将根据两者的比较结果进行最小单位地修改视图，而不是将整个视图根据新的VNode重绘。patch的核心在于diff算法，这套算法可以高效地比较Virtual DOM的变更，得出变化以修改视图

> 太长不看: patch方法只修改前后有差异的VNode