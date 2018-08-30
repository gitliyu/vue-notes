# Observer与数据驱动原理
Observer是Vue很核心的一个功能，是实现数据双向绑定的关键，这一篇就对这部分做一下介绍，对应源码文件夹`src/core/observer`

参考： ['vue2.0-source'](https://github.com/liutao/vue2.0-source)

首先需要说明的是，Observer依赖于['Object.defineProperty()'](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty)方法，这也是Vue不支持ie9以下浏览器的原因，关于这个方法就不进行介绍了
数据驱动实现的基本原理就是，通过对象属性的get方法设置观察者，在数据变化也就是set方法触发时更新虚拟dom结构，在下次tick驱动视图更新。
主要分为以下三部分：
1. Observer: Vue初始化时调用，递归地为对象所有属性设置setter/getter
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
        // 判断属性为为对象后递归
        if (value && typeof value === 'object') {
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
                    // 当然，实际上添加的并不是相同的Dep.target
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
可以看到，在Observer初始化时，首先会创建一个对应的Dep，之后会递归的为所有对象属性设置setter/getter。

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
    this.notify = function(){
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
        fn();
        Dep.target = null;
    }
    this.update();
}
```
Watcher内部的更新方法被触发时，会将自身存放在Dep上作为当前响应的Watcher，回调结束后清空
自运行一次update方法是为了在最初渲染模板过程中，调用数据对象的getter时建立两者之间的关系，否则Watcher不会生效。
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
    new Watcher(function(){
        document.querySelector("#box").innerHTML = obj.a;
    })
</script>
```
接下来就可以调用Observer方法对对象属性进行设置了，执行之后将obj对象打印出来，可以看到如下结果
!['Observer设置后的obj对象'](https://github.com/gitliyu/vue-notes/blob/master/images/observer.png)
那么如何去验证方法是否生效呢，打开控制台改变obj对象的属性，比如我们输入`obj.a = 2`，而可以看到如下结果
!['结果'](https://github.com/gitliyu/vue-notes/blob/master/images/observer-result.png),这样就大概的实现了数据驱动视图，接下来对源码进行分析。

> 未完待续