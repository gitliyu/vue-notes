# Observer与数据驱动原理
Observer是Vue很核心的一个功能，是实现数据双向绑定的关键，这一篇就对这部分做一下介绍，对应源码文件夹`src/core/observer`

参考： ['vue2.0-source'](https://github.com/liutao/vue2.0-source)

首先需要说明的是，Observer依赖于['Object.defineProperty()'](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty)方法，这也是Vue不支持ie9以下浏览器的原因，关于这个方法可以点击查看文档

数据驱动实现的基本原理就是，通过对象属性的get方法设置观察者，在数据变化也就是set方法触发时更新虚拟dom结构，在下次tick驱动视图更新

主要分为以下三部分：
1. Observer: Vue初始化时调用，递归地为data，props，computed对象所有属性设置setter/getter
2. Watcher: 观察者，当监听的数据值修改时，执行响应的回调函数
3. Dep: 每一个Observer对应一个Dep，作为Observer和所有Watcher通信的中间件，内部保存与该Observer相关的Watcher

下面介绍源码之前，先来一个简单的Demo['点击查看完整代码'](https://github.com/gitliyu/vue-notes/blob/master/example/observer.html)：

```javascript
function Observer(obj){
    // 设置对应的Dep
    let dep = new Dep();

    // 遍历所有对象属性设置setter/getter
    Object.keys(obj).forEach( key => {
        let value = obj[key];
        // 判断属性为对象后递归
        if (value && typeof value === 'object' && !Array.isArray(value)) {
            Object.keys(value).forEach( index => {
                new Observer(value);
            })
        };

        Object.defineProperty(obj, key, {
            enumerable: true,  
            configurable: true,
            // 设置watcher
            get: () => {
                // 如果当前正有Watcher触发，将Watcher添加到队列中
                // Dep方法会在下面介绍
                if (Dep.target) {
                    // Watcher收集
                    dep.addSub(Dep.target);
                };
                return value;
            },
            // 触发响应变化
            set: val => {
                value = val;
                dep.notify();
            }
        })
    })
}
```
可以看到，在Observer初始化时，首先会创建一个对应的Dep，之后会递归的为所有对象属性设置setter/getter

```javascript
function Dep(){
    // 存放当前触发的Watcher
    this.target = null;
    // 存放watcher队列
    this.subs = [];

    // 向队列中添加Watcher
    this.addSub = watcher => {
        this.subs.push(watcher);
    }

    // 通知Watcher
    this.notify = () => {
        this.subs.forEach( watcher => {
            watcher.update();
        });
    }
}
```
Dep内部对象的target为当前Watcher，同时自身也有一个数组属性用来存放Watcher队列

```javascript
function Watcher(fn){
    // 响应方法，触发回调
    this.update = () => {
        Dep.target = this;
        // 回调调用时会触发obj.a的get方法设置为其Watcher
        fn();
        Dep.target = null;
    }
    this.update();
}
```
Watcher内部的更新方法被触发时，会将自身存放在Dep上作为当前响应的Watcher，回调结束后清空

自运行一次update方法是为了在最初渲染模板过程中，调用数据对象时触发get方法在Dep队列中增加对应的Watcher
```javascript
<body>
    <div id="box"></div>		
</body>
</html>
<script type="text/javascript">
    var obj = {
        a: 1,
        b: 2,
        c: 3
    }

    new Observer(obj);
    new Watcher(() => {
        document.querySelector("#box").innerHTML = obj.a;
    })
</script>
```
接下来就可以调用Observer方法对对象属性进行设置了，执行之后将obj对象打印出来，可以看到如下结果

!['Observer设置后的obj对象'](https://github.com/gitliyu/vue-notes/blob/master/images/observer.png)

那么如何去验证方法是否生效呢，打开控制台改变obj对象的属性，比如我们输入`obj.a = 2`，而可以看到如下结果

!['结果'](https://github.com/gitliyu/vue-notes/blob/master/images/observer-result.png)

这样就大概的实现了数据驱动视图，在了解了原理之后，接下来对vue源码中的使用进行分析。

### Init阶段
之前在Vue实例化过程的介绍中提到过，在实例化时会首先调用`_init`方法进行初始化，其中执行了`initState`方法对data，props，computed属性等进行了设置，我们来到`src/core/instance/state.js`
```javascript
export function initState (vm: Component) {
  vm._watchers = []
  const opts = vm.$options
  if (opts.props) initProps(vm, opts.props)
  if (opts.methods) initMethods(vm, opts.methods)
  if (opts.data) {
    initData(vm)
  } else {
    observe(vm._data = {}, true /* asRootData */)
  }
  if (opts.computed) initComputed(vm, opts.computed)
  if (opts.watch && opts.watch !== nativeWatch) {
    initWatch(vm, opts.watch)
  }
}
```
以data为例，找到`initData`方法
```javascript
function initData (vm: Component) {
  let data = vm.$options.data
  // data的两种定义方式
  data = vm._data = typeof data === 'function'
    ? getData(data, vm)
    : data || {}
  ……
  // 循环将data注册在vm上
  const keys = Object.keys(data)
  const props = vm.$options.props
  const methods = vm.$options.methods
  let i = keys.length
  while (i--) {
    const key = keys[i]
    if (process.env.NODE_ENV !== 'production') {
      if (methods && hasOwn(methods, key)) {
        warn(
          `Method "${key}" has already been defined as a data property.`,
          vm
        )
      }
    }
    if (props && hasOwn(props, key)) {
      process.env.NODE_ENV !== 'production' && warn(
        `The data property "${key}" is already declared as a prop. ` +
        `Use prop default value instead.`,
        vm
      )
    } else if (!isReserved(key)) {
      proxy(vm, `_data`, key)
    }
  }
  // 设置响应式属性
  observe(data, true /* asRootData */)
}
```
可以看到，这个方法主要做了两件事
- 在`options`中拿到data属性后，通过使用`proxy`把每一个值`vm._data.xxx`都代理到`vm.xxx`上
- 使用`observe`方法将所有data属性设置为响应式属性

```javascript
export function proxy (target: Object, sourceKey: string, key: string) {
  sharedPropertyDefinition.get = function proxyGetter () {
    return this[sourceKey][key]
  }
  sharedPropertyDefinition.set = function proxySetter (val) {
    this[sourceKey][key] = val
  }
  Object.defineProperty(target, key, sharedPropertyDefinition)
}
```
`proxy`方法通过`Object.defineProperty`将`target[sourceKey][key]`的读写修改为`target[key]`，所以我们可以通过`this.xxx`访问到`this._data.xxx`

### Observer
首先介绍一下`observe`这个方法，来到文件`src/core/observer/index.js`
```javascript
export function observe (value: any, asRootData: ?boolean): Observer | void {
  // 判断是vnode对象就返回 
  if (!isObject(value) || value instanceof VNode) {
    return
  }
  let ob: Observer | void
  // 判断已经是响应式属性
  if (hasOwn(value, '__ob__') && value.__ob__ instanceof Observer) {
    ob = value.__ob__
  } else if (
    shouldObserve &&  // Observe状态
    !isServerRendering() &&     // 非服务端渲染
    (Array.isArray(value) || isPlainObject(value)) &&   // 是非空非数组的纯对象
    Object.isExtensible(value) &&   // 可扩展的
    !value._isVue   // 不是vue对象
  ) {
    ob = new Observer(value)
  }
  if (asRootData && ob) {
    ob.vmCount++
  }
  return ob
}
```
`observe`方法会尝试创建一个Observer实例（__ob__），如果成功创建会返回一个Observer实例，或者返回已有的Observer实例，主要是进行了一系列的判断
```javascript
export class Observer {
  value: any;
  dep: Dep;
  vmCount: number; // number of vms that has this object as root $data

  constructor (value: any) {
    this.value = value
    // 设置对应的Dep
    this.dep = new Dep()
    this.vmCount = 0
    // 在对象上定义'__ob__'属性，之前说过observe方法会对其进行检查
    def(value, '__ob__', this)
    // 对数组和对象分别进行设置
    if (Array.isArray(value)) {
      const augment = hasProto
        ? protoAugment
        : copyAugment
      augment(value, arrayMethods, arrayKeys)
      this.observeArray(value)
    } else {
      this.walk(value)
    }
  }

  // 遍历每一个对象属性并且在它们上面绑定getter与setter
  walk (obj: Object) {
    const keys = Object.keys(obj)
    for (let i = 0; i < keys.length; i++) {
      defineReactive(obj, keys[i])
    }
  }
  // 对一个数组的每一个元素进行observe
  observeArray (items: Array<any>) {
    for (let i = 0, l = items.length; i < l; i++) {
      observe(items[i])
    }
  }
}
```
Observer的实例方法会对类型进行判断，当判断类型为数组时使用`observeArray`方法，对每一个数组元素来创建Observer实例，当判断类型为对象时，使用`walk`方法，遍历对象属性后执行`defineReactive`方法
```javascript
export function defineReactive (
  obj: Object,
  key: string,
  val: any,
  customSetter?: Function
) {
  const dep = new Dep()

  // 拿到obj的属性描述符
  const property = Object.getOwnPropertyDescriptor(obj, key)
  // 如果描述符无法改变，返回
  if (property && property.configurable === false) {
    return
  }

  // 如果之前该对象已经预设了getter以及setter函数则将其取出来
  // 新定义的getter/setter中会将其执行，保证不会覆盖之前已经定义的getter/setter
  const getter = property && property.get
  const setter = property && property.set

  // 为子属性执行observe，相当于递归的对所有属性进行设置，并返回子节点Observer对象
  let childOb = observe(val)
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter () {
      // 使用已有getter
      const value = getter ? getter.call(obj) : val
      if (Dep.target) {
        // 自身以及子对象的依赖收集
        dep.depend()
        if (childOb) {
          childOb.dep.depend()
        }
        if (Array.isArray(value)) {
          dependArray(value)
        }
      }
      return value
    },
    set: function reactiveSetter (newVal) {
      const value = getter ? getter.call(obj) : val
      /* eslint-disable no-self-compare */
      if (newVal === value || (newVal !== newVal && value !== value)) {
        return
      }
      if (process.env.NODE_ENV !== 'production' && customSetter) {
        customSetter()
      }
      if (setter) {
        setter.call(obj, newVal)
      } else {
        val = newVal
      }
      // 新的值需要重新进行observe，保证数据响应式
      childOb = observe(newVal)
      // dep对象通知所有的观察者
      dep.notify()
    }
  })
}
```
这个方法就是最终对对象设置setter/getter的地方，也就是demo中Observer方法做的工作

首先创建对应的`dep`，拿到对象自身的属性描述符，如果已经有setter/getter，在设置时会作为默认属性，而不会将其覆盖
使用observe方法实现对对象属性的递归设置，保证所有子属性都被覆盖到，之后通过`Object.defineProperty`重新定义对象属性描述符

### Dep
打开文件`src/core/observer/dep.js`
```javascript
let uid = 0

export default class Dep {
  // 同时只能由唯一的Watcher被触发
  static target: ?Watcher;
  // 作为dep的唯一标识
  id: number;
  // 保存Watcher的数组
  subs: Array<Watcher>;

  constructor () {
    this.id = uid++
    this.subs = []
  }

  // 添加一个Watcher对象 
  addSub (sub: Watcher) {
    this.subs.push(sub)
  }
  
  // 移除一个观察者对象
  removeSub (sub: Watcher) {
    remove(this.subs, sub)
  }

  // 依赖收集，当存在Dep.target的时候添加观察者对象
  depend () {
    if (Dep.target) {
      Dep.target.addDep(this)
    }
  }

  // 通知依赖subs中的所有Watcher
  notify () {
    const subs = this.subs.slice()
    if (process.env.NODE_ENV !== 'production' && !config.async) {
      subs.sort((a, b) => a.id - b.id)
    }
    for (let i = 0, l = subs.length; i < l; i++) {
      subs[i].update()
    }
  }
}

// 依赖收集完需要将Dep.target设为null，防止后面重复添加依赖
Dep.target = null
const targetStack = []

// 将watcher观察者实例设置给Dep.target，用以依赖收集。同时将该实例存入target栈中
export function pushTarget (_target: ?Watcher) {
  if (Dep.target) targetStack.push(Dep.target)
  Dep.target = _target
}

// 将观察者实例从target栈中取出并设置给Dep.target
export function popTarget () {
  Dep.target = targetStack.pop()
}
```

### Watcher

> 未完待续
