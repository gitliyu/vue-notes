# 前端路由实现：Hash和History模式

前端路由需要实现的核心功能就是更新视图但不重新请求页面，目前在浏览器环境中这一功能的实现主要有两种方式：
1. URL中的`hash`("#")
2. HTML5中新增的`History API`

> todo