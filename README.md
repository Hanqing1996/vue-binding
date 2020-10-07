参考：https://juejin.im/entry/6844903479044112391



根据视图更新数据，只要给元素添加事件就能做到。因此我们这里只着重于实现“数据变化后更新视图”。



#### 重要概念

* Observer

  监听属性变化，并通知 Watcher

* Watcher

  根据属性变化执行相应函数

* Dep

  专门收集 watcher

---

#### 实现一个Observer

> 递归监听一个对象的所有属性/子属性

```js
function defineReactive(data, key, val) {
    observe(val); // 递归遍历所有子属性
    Object.defineProperty(data, key, {
        enumerable: true,
        configurable: true,
        get: function() {
            return val;
        },
        set: function(newVal) {
            val = newVal;
            console.log('属性' + key + '已经被监听了，现在值为：“' + newVal.toString() + '”');
        }
    });
}
 
function observe(data) {
    if (!data || typeof data !== 'object') {
        return;
    }
    Object.keys(data).forEach(function(key) {
        defineReactive(data, key, data[key]);
    });
};
 
var library = {
    book1: {
        name: ''
    },
    book2: ''
};
observe(library);
library.book1.name = 'vue权威指南'; // 属性name已经被监听了，现在值为：“vue权威指南”
library.book2 = '没有此书籍';  // 属性book2已经被监听了，现在值为：“没有此书籍”
```

#### 实现 Dep

> 收集 watcher（subs）,并在被监听属性变化时通知（notify） watcher

```js
function defineReactive(data, key, val) {
    observe(val); // 递归遍历所有子属性
    var dep = new Dep(); 
    Object.defineProperty(data, key, {
        enumerable: true,
        configurable: true,
        get: function() {
            if (是否需要添加订阅者) {
                dep.addSub(watcher); // 在这里添加一个订阅者
            }
            return val;
        },
        set: function(newVal) {
            if (val === newVal) {
                return;
            }
            val = newVal;
            console.log('属性' + key + '已经被监听了，现在值为：“' + newVal.toString() + '”');
            dep.notify(); // 如果数据变化，通知所有订阅者
        }
    });
}
 
function Dep () {
    this.subs = [];
}
Dep.prototype = {
    addSub: function(sub) {
        this.subs.push(sub);
    },
    notify: function() {
        this.subs.forEach(function(sub) {
            sub.update();
        });
    }
};
```

#### 实现 Watcher

> 一个 Watcher 实例需要指明它所监听的属性（exp），属性变化时要执行的函数（cb）,并同步被监听属性的最新值（value） 。在它被创建时，会被添加到 Dep 中。每次 被监听的某个属性变化后，Dep 会通知（notify）所有 Watcher 实例，它们会自行检查（update）是不是自己负责监听的属性发生了变化，从而判断要不要执行相应函数。

我们只要在订阅者Watcher初始化的时候才需要添加订阅者，所以需要做一个判断操作，因此可以在订阅器上做一下手脚：在Dep.target上缓存下订阅者，添加成功后再将其去掉就可以了。订阅者Watcher的实现如下：

```js
function Watcher(vm, exp, cb) {
    this.cb = cb;
    this.vm = vm;
    this.exp = exp;
    this.value = this.get();  // 将自己添加到订阅器的操作
}
 
Watcher.prototype = {
    update: function() {
        this.run();
    },
    run: function() {
        var value = this.vm.data[this.exp];
        var oldVal = this.value;
        if (value !== oldVal) {
            this.value = value;
            this.cb.call(this.vm, value, oldVal);
        }
    },
    get: function() {
        Dep.target = this;  // 缓存自己
        var value = this.vm.data[this.exp]  // 强制执行监听器里的get函数
        Dep.target = null;  // 释放自己
        return value;
    }
};
```

defineReactive 函数也做相应调整，以保证在一个 Watcher 实例被创建时，Dep 会收集该实例

 ```js
function defineReactive(data, key, val) {
    observe(val); // 递归遍历所有子属性
    var dep = new Dep(); 
    Object.defineProperty(data, key, {
        enumerable: true,
        configurable: true,
        get: function() {
            if (Dep.target) {.  // 判断是否需要添加订阅者
                dep.addSub(Dep.target); // 在这里添加一个订阅者
            }
            return val;
        },
        set: function(newVal) {
            if (val === newVal) {
                return;
            }
            val = newVal;
            console.log('属性' + key + '已经被监听了，现在值为：“' + newVal.toString() + '”');
            dep.notify(); // 如果数据变化，通知所有订阅者
        }
    });
}
Dep.target = null;
 ```

---

#### 实现 selfVue

> 设置要监听哪些属性，并将状态与DOM节点挂钩

```html
<body>
    <h1 id="name">{{name}}</h1>
</body>
```

```js
function SelfVue (data, el, exp) {
    this.data = data;
    observe(data);
    el.innerHTML = this.data[exp];  // 初始化模板数据的值
    new Watcher(this, exp, function (value) {
        // 这里是属性变化后的执行函数，这里为了简单起见，设置属性值变化后，el的 innerHTML 也同步变化
        el.innerHTML = value; 
    });
    return this;
}
```

**到此为止，selfVue 已经具有了“在数据变化后，更新视图”的功能。** 完整代码戳此处：https://github.com/canfoo/self-vue/tree/master/v1

```html
<body>
    <h1 id="name">{{name}}</h1>
</body>
<script src="js/observer.js"></script>
<script src="js/watcher.js"></script>
<script src="js/index.js"></script>
<script type="text/javascript">
    var ele = document.querySelector('#name');
    var selfVue = new SelfVue({
        name: 'hello world'
    }, ele, 'name');
 
    window.setTimeout(function () {
        console.log('name值改变了');
        selfVue.data.name = 'canfoo';
    }, 2000);
 
</script>
```

---

#### 实现 Compile

Compile 只在最开始被调用一次，它负责

1. 查询模板中哪些节点要做数据绑定，对于要绑定的节点，自动添加 watcher 实例

2. 依据 selfVue 的 data 设置视图的 innerHTML 初始值

```js
function Compile(el, vm) {
    this.vm = vm;
    this.el = document.querySelector(el);
    this.fragment = null;
    this.init();
}

Compile.prototype = {
    init: function () {
        if (this.el) {
            this.fragment = this.nodeToFragment(this.el);
            this.compileElement(this.fragment);
            this.el.appendChild(this.fragment);
        } else {
            console.log('Dom元素不存在');
        }
    },
    nodeToFragment: function (el) {
        var fragment = document.createDocumentFragment();
        var child = el.firstChild;
        while (child) {
            // 将Dom元素移入fragment中
            fragment.appendChild(child);
            child = el.firstChild
        }
        return fragment;
    },
    compileElement: function (el) {
        var childNodes = el.childNodes;
        var self = this;
        [].slice.call(childNodes).forEach(function(node) {
            var reg = /\{\{\s*(.*?)\s*\}\}/;
            var text = node.textContent;
            if (self.isTextNode(node) && reg.test(text)) {  // 判断是否是符合这种形式{{}}的指令
                self.compileText(node, reg.exec(text)[1]);
            }

            if (node.childNodes && node.childNodes.length) {
                self.compileElement(node);  // 继续递归遍历子节点
            }
        });
    },
    compileText: function(node, exp) {
        var self = this;
        var initText = this.vm[exp];
        this.updateText(node, initText);  // 将初始化的数据初始化到视图中
        new Watcher(this.vm, exp, function (value) { // 生成订阅器并绑定更新函数
            self.updateText(node, value);
        });
    },
    updateText: function (node, value) {
        node.textContent = typeof value == 'undefined' ? '' : value;
    },
    isTextNode: function(node) {
        return node.nodeType == 3;
    }
}
```

优于 Watcher 实例的添加在 Compile 中已经实现，所以 selfVue 只要调用一次 Compile，即可完成依赖的全部收集

#### selfVue

```js
function SelfVue (options) {
    var self = this;
    this.vm = this;
    this.data = options.data;
    
    // 这句代码主要是做个代理，使得访问 self.name 等效于访问 self.
    Object.keys(this.data).forEach(function(key) {
        self.proxyKeys(key);
    });

    observe(this.data);
    new Compile(options.el, this.vm); // 直接调用 Compile，让 Compile 去完成要绑定数据的节点查询，并赋予初值。
    return this;
}

SelfVue.prototype = {
    proxyKeys: function (key) {
        var self = this;
        Object.defineProperty(this, key, {
            enumerable: false,
            configurable: true,
            get: function proxyGetter() {
                return self.data[key];
            },
            set: function proxySetter(newVal) {
                self.data[key] = newVal;
            }
        });
    }
}
```

然后，我们直接这么用。完整代码戳此：https://github.com/canfoo/self-vue/tree/master/v2

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>self-vue</title>
</head>
<style>
    #app {
        text-align: center;
    }
</style>
<body>
    <div id="app">
        <h2>{{ title }}</h2>
        <h1>{{ name }}</h1>
    </div>
</body>
<script src="js/observer.js"></script>
<script src="js/watcher.js"></script>
<script src="js/compile.js"></script>
<script src="js/index.js"></script>

<script type="text/javascript">

    var selfVue = new SelfVue({
		// 查询 #app 的子元素有无要绑定数据的节点
        el: '#app',
        data: {
            title: 'hello world',
            name: ''
        }
    });

    window.setTimeout(function () {
        selfVue.title = '你好';
    }, 2000);

    window.setTimeout(function () {
        selfVue.name = 'canfoo';
    }, 2500);

</script>
</html>
```

---

此外，可以进一步扩展 Compile，让它可以识别带有各类指令的节点。

```html
<body>
    <div id="app">
        <h2>{{title}}</h2>
        <input v-model="name">
        <h1>{{name}}</h1>
        <button v-on:click="clickMe">click me!</button>
    </div>
</body>
```

完整代码戳此：https://github.com/canfoo/self-vue/tree/master/v3
