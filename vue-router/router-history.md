# 前端路由实现：Hash和History模式

前端路由需要实现的核心功能就是更新视图但不重新请求页面，目前在浏览器环境中这一功能的实现主要有两种方式：
1. URL中的`hash`("#")
2. HTML5中新增的`History API`

首先来回顾一下`VueRouter`在实例化时对于路由模式的定义规则，具体可以查看['VueRouter类介绍'](https://github.com/gitliyu/vue-notes/blob/master/vue-router/router-define.md)
1. 作为参数传入的字符串属性`mode`，对应着不同的`history`的实现类，两者对应关系如下：
	- history -> HTML5History
	- hash -> HashHistory
	- abstract -> AbstractHistory
2. 在初始化对应的`history`之前，会对`mode`做一些校验：默认为'hash'；若浏览器不支持`HTML5History`（通过supportsPushState变量判断），则`mode`强制设为'hash'；若不是在浏览器环境下运行，则`mode`强制设为'abstract'
3. `VueRouter`类中的`push()`, `replace()`等方法只是一个代理，实际是调用的具体`history`对象的对应方法，在`init()`方法中初始化时，也是根据`history`对象具体的类别执行不同操作

> todo