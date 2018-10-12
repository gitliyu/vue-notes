# Vue实例属性
到了这一篇，基本上Vue原型上定义的大部分属性都认识了，就在这里进行一下归纳总结吧

`Vue`实例是以组件的形式展现的，想要看到所有实例属性，可以找到对于`component`这一类型的`Flow`定义，位于`flow/component.js`，这里只贴出相关代码
```javascript
declare interface Component {
  ……

  // 供开发者使用的公共属性
  $el: any; // 组件对应的元素
  $data: Object;	// 数据
  $props: Object;	// 传入的props数据
  $options: ComponentOptions;	// 传入的options
  $parent: Component | void;	// 父节点
  $root: Component;	  // 根节点
  $children: Array<Component>;	// 子节点
  $refs: { [key: string]: Component | Element | Array<Component | Element> | void };	// ref节点集合
  $slots: { [key: string]: Array<VNode> };	// 所有插槽
  $scopedSlots: { [key: string]: () => VNodeChildren };	// 作用于插槽
  $vnode: VNode; // 当前自定义组件在父组件中的vnode，等同于vm.$options._parentVnode
  $attrs: { [key: string] : string };	// 非props的绑定属性
  $listeners: { [key: string]: Function | Array<Function> };	// 事件监听
  $isServer: boolean;	// 是否服务端渲染

  // 供开发者使用的公共方法
  $mount: (el?: Element | string, hydrating?: boolean) => Component;
  $forceUpdate: () => void;
  $destroy: () => void;
  $set: <T>(target: Object | Array<T>, key: string | number, val: T) => T;
  $delete: <T>(target: Object | Array<T>, key: string | number) => void;
  $watch: (expOrFn: string | Function, cb: Function, options?: Object) => Function;
  $on: (event: string | Array<string>, fn: Function) => Component;
  $once: (event: string, fn: Function) => Component;
  $off: (event?: string | Array<string>, fn?: Function) => Component;
  $emit: (event: string, ...args: Array<mixed>) => Component;
  $nextTick: (fn: Function) => void | Promise<*>;
  $createElement: (tag?: string | Component, data?: Object, children?: VNodeChildren) => VNode;

  // 私有属性
  _uid: number | string;	// 标识符，唯一的自增id
  _name: string; // 组件名
  _isVue: true;	 // 标识为vue对象，避免被重复observe
  _self: Component;	// 当前vm实例
  _renderProxy: Component;	// Proxy代理对象
  _renderContext: ?Component;
  _watcher: Watcher;	// 当前watcher
  _watchers: Array<Watcher>;	
  _computedWatchers: { [key: string]: Watcher };	// 保存计算属性创建的watcher对象
  _data: Object;	// 响应式data属性
  _props: Object;	// 响应式props属性
  _events: Object;	// 当前元素上绑定的自定义事件
  _inactive: boolean | null;
  _directInactive: boolean;
  _isMounted: boolean;	// 标识已经渲染完成
  _isDestroyed: boolean;	// 标识已经被销毁
  _isBeingDestroyed: boolean;	// 标识开始被销毁
  _vnode: ?VNode; // 自身节点
  _staticTrees: ?Array<VNode>; // 当前组件模板内分析出的静态内容的render函数数组
  _hasHookEvent: boolean;	// 标示是否生命周期钩子函数，即hook事件
  _provided: ?Object;

  // 生命周期相关
  _init: Function;	// 初始化方法，在initMixin中被定义
  _mount: (el?: Element | void, hydrating?: boolean) => Component;	// 渲染方法，执行mountComponent
  _update: (vnode: VNode, hydrating?: boolean) => void;  // 更新方法，在lifecycleMixin中被定义

  // 渲染函数
  _render: () => VNode;  // 创建VNode

  __patch__: (	// 为新旧VNode执行patch
    a: Element | VNode | void,
    b: VNode,
    hydrating?: boolean,
    removeOnly?: boolean,
    parentElm?: any,
    refElm?: any
  ) => any;

  ……
};
```
接下来看一下`vm.$options`上面的属性，除了少部分私有属性外，其他都是我们在创建`vue`实例时传入的数据，同样找到这一类型的`Flow`定义，位于`flow/options.js`
```javascript
declare type ComponentOptions = {
  componentId?: string;

  // 定义的数据和方法
  data: Object | Function | void;
  props?: { [key: string]: PropOptions };
  propsData?: ?Object;
  computed?: {
    [key: string]: Function | {
      get?: Function;
      set?: Function;
      cache?: boolean
    }
  };
  methods?: { [key: string]: Function };
  watch?: { [key: string]: Function | string };

  // DOM
  el?: string | Element;	// 对应的元素
  template?: string;	// 模板字符串
  render: (h: () => VNode) => VNode;	// render方法
  renderError?: (h: () => VNode, err: Error) => VNode;
  staticRenderFns?: Array<() => VNode>;

  // 生命周期钩子函数
  beforeCreate?: Function;
  created?: Function;
  beforeMount?: Function;
  mounted?: Function;
  beforeUpdate?: Function;
  updated?: Function;
  activated?: Function;
  deactivated?: Function;
  beforeDestroy?: Function;
  destroyed?: Function;
  errorCaptured?: () => boolean | void;

  // 指令，子组件，过渡动画，过滤器
  directives?: { [key: string]: Object };
  components?: { [key: string]: Class<Component> };
  transitions?: { [key: string]: Object };
  filters?: { [key: string]: Function };

  // context
  provide?: { [key: string | Symbol]: any } | () => { [key: string | Symbol]: any };
  inject?: { [key: string]: InjectKey | { from?: InjectKey, default?: any }} | Array<string>;

  // 组件v-model
  model?: {
    prop?: string;
    event?: string;
  };

  // misc
  parent?: Component;	// 父组件实例
  mixins?: Array<Object>;	// mixins混入的数据
  name?: string;	// 组建名
  extends?: Class<Component> | Object;	// extends传入的数据
  delimiters?: [string, string];	// 模板分隔符
  comments?: boolean;
  inheritAttrs?: boolean;

  // 私有属性
  _isComponent?: true;  // 是否是组件
  _propKeys?: Array<string>; // props传入对象的键数组
  _parentVnode?: VNode; // 当前组件，在父组件中的VNode对象
  _parentListeners?: ?Object; // 当前组件，在父组件上绑定的事件
  _renderChildren?: ?Array<VNode>; // 父组件中定义在当前元素内的子元素的VNode数组（slot）
  _componentTag: ?string;  // 自定义标签名
  _scopeId: ?string;
  _base: Class<Component>; // Vue
};
```
