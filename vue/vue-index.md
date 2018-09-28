# 源码目录结构
分析`Vue`源码的第一步，先了解源码中目录的设计和各个模块的功能，对应`Vue`版本为`2.5.17`，以后会对目录进行补充。

参考： ['vue2.0-source'](https://github.com/liutao/vue2.0-source)  ['vue-analysis'](https://github.com/ustbhuangyi/vue-analysis)
- **dist** - 打包文件目录
- **examples** - 示例
- **flow** - 定义`Flow`静态类型
- **packages** - `vue`生成的其它的`npm`包
- **script** - 打包配置文件
- **src** - 主要源码所在位置
	 - **compiler** - 编译相关，模板解析的所有文件，将`teamplate`转为`render`函数
	 	- **codegen** - 编译生成`render`函数
	 	- **directives** - 通用生成`render`函数之前需要处理的指令
	 	- **parser** - 模板解析，将模板字符串转换为元素抽象语法树
	 - **core** - 核心代码
	 	- **components** - 内置组件
	 	- **global-api** -  全局Api方法的封装，比如`Vue.use`,`Vue.extend`,`Vue.mixin`等
		- **instance** - 实例相关内容，包括实例方法，生命周期，事件等
		- **observer** - 响应式数据，观察者
		- **util** - 工具函数
		- **vdom** - 虚拟dom `vnode`相关
	- **platforms** - 平台相关的内容
		- **web** - web端相关文件
			- **compiler** - 编译阶段需要处理的指令和模块
			- **runtime** - 运行阶段需要处理的组件、指令和模块
			- **server** - 服务端渲染相关
			- **util** - 工具库
		- **weex** - weex端相关文件
	- **server** - 服务端渲染相关
	- **sfc** - 解析，通过使用`webpack`将`.vue`文件内容解析
	- **shared** - 共享的工具方法
 - **test** - 测试文件
 - **types** - `typescript`相关
