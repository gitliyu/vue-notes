# 路由匹配
在上文['VueRouter类介绍'](https://github.com/gitliyu/vue-notes/blob/master/vue-router/router-define.md)可以看到对路由匹配的处理
```javascript
this.matcher = createMatcher(options.routes || [], this)
```
将传入的routes配置数组进行处理，得到路由匹配的详细信息，本文主要介绍一下`createMatcher`方法做了什么，来到`src/create-matcher.js`
```javascript
import { createRoute } from './util/route'
import { createRouteMap } from './create-route-map'

export function createMatcher (
  routes: Array<RouteConfig>,
  router: VueRouter
): Matcher {
  const { pathList, pathMap, nameMap } = createRouteMap(routes)

  // 将createRouteMap方法封装，以供vueRouter调用
  function addRoutes (routes) {
    createRouteMap(routes, pathList, pathMap, nameMap)
  }

  // 核心的match方法
  function match (
    raw: RawLocation,
    currentRoute?: Route,
    redirectedFrom?: Location
  ): Route {
  	// 获取路由name
    const location = normalizeLocation(raw, currentRoute, false, router)
    const { name } = location

    if (name) {
      // 获取路由记录 
      const record = nameMap[name]
      if (process.env.NODE_ENV !== 'production') {
        warn(record, `Route with name '${name}' does not exist`)
      }
      // 无记录，返回创建 route
      if (!record) return _createRoute(null, location)
      // 路由参数的处理
      const paramNames = record.regex.keys
        .filter(key => !key.optional)
        .map(key => key.name)

      if (typeof location.params !== 'object') {
        location.params = {}
      }

      if (currentRoute && typeof currentRoute.params === 'object') {
        for (const key in currentRoute.params) {
          if (!(key in location.params) && paramNames.indexOf(key) > -1) {
            location.params[key] = currentRoute.params[key]
          }
        }
      }

      // 有记录，返回创建 route
      if (record) {
        location.path = fillParams(record.path, location.params, `named route "${name}"`)
        return _createRoute(record, location, redirectedFrom)
      }
    } else if (location.path) {
      // 普通路由处理
      location.params = {}
      for (let i = 0; i < pathList.length; i++) {
        const path = pathList[i]
        const record = pathMap[path]
        if (matchRoute(record.regex, location.path, location.params)) {
          return _createRoute(record, location, redirectedFrom)
        }
      }
    }
    // 没有匹配路由时，按照空记录进行处理
    return _createRoute(null, location)
  }

  function redirect () { …… }
  function alias () { …… }

  //  创建路由route
  function _createRoute (
    record: ?RouteRecord,
    location: Location,
    redirectedFrom?: Location
  ): Route {
  	// 重定向和别名逻辑，分别调用redirect和alias方法进行处理
    if (record && record.redirect) {
      return redirect(record, redirectedFrom || location)
    }
    if (record && record.matchAs) {
      return alias(record, location, record.matchAs)
    }
    // 最终调用的是util中的createRoute方法创建route
    return createRoute(record, location, redirectedFrom, router)
  }

  return {
    match,
    addRoutes
  }
}
```
简单总结一下以上代码，根据传入的`routes`生成路由匹配表，并且返回`match`函数以及一个可以增加路由配置项`addRoutes`函数，以供`VueRouter`类调用

> 未完待续