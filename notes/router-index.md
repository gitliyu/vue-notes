# 源码目录结构
先来了解下源码中目录的设计和各个模块的功能，相比`Vue`来看十分简单，对应`vue-router`版本为`3.0.1`

- **build** - 打包配置文件
- **dist** - 打包文件目录
- **docs** - 文档
- **examples** - 示例
- **flow** - 定义Flow静态类型
- **src** - 主要源码所在位置
	 - **components** - 定义`router-view`和`router-link`组件
	 - **history** - 路由方式的封装方法
	 - **util** - 各种功能类和功能函数
	 - **create-matcher** - 生成匹配表
	 - **create-router-mapr** - 生成匹配表
	 - **index** - 定义`VueRouter`，入口文件
	 - **Install** - 提供安装方法
 - **test** - 测试文件
 - **types** - typescript相关