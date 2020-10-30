> 最近参加了很多场面试，几乎每场面试中都会问到Vue源码方面的问题。在此开一个系列的专栏，来总结一下这方面的经验，如果觉得对您有帮助的，不妨点个赞支持一下呗。

## 前言

Vue有两个核心思想，一个是数据驱动，简单来说就是通过模板和数据渲染成最终的 DOM ，具体是如何实现在上一篇[🚩Vue源码——模板和数据如何渲染成最终的DOM](https://juejin.im/post/6871161430830235662#heading-0)中详细地介绍过了。

另外一个是组件化，谓组件化，就是把一个页面拆分成多个组件，这些组件是独立的，可复用的，可嵌套的，等这些组件开发完成后，像搭积木一样拼装成一个页面。

本文会在[上一篇](https://juejin.im/post/6871161430830235662#heading-0)的基础上来详细介绍在 Vue 中组件如何渲染成最终的 DOM ,其过程与通过模板和数据渲染成最终的 DOM 有何不同。

先创建一个简单的 demo ，基于这个设定的场景来研究。
```
<!DOCTYPE html>
<html>
    <head>
        <script src="https://cdn.jsdelivr.net/npm/vue/dist/vue.js"></script>
    </head>
    <body>
        <div id="app"></div>
    </body>
    <script>
        const aa ={
          template:'<div><p>{{aa}}<span>{{bb}}</span></p></div>',
          data(){
              return{
                  aa:'欢迎',
                  bb:'Vue'
              }
          }
        }
        var app = new Vue({
            el: '#app',
            render: h =>h(aa)
        })
    </script>
</html>
```
回顾上一篇模板和数据渲染成最终的 DOM 的逻辑流程图
![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1ee367ccf60f4d10a2980328e04b49d7~tplv-k3u1fbpfcp-zoom-1.image)
跟组件渲染成最终的 DOM 的逻辑流程图对比。
![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c0a8b0ec442142cb860da4cf8266aacd~tplv-k3u1fbpfcp-zoom-1.image)
从`new Vue()` 到`new Watcher()`过程都是一样的。因为在上一篇 demo 中 render 方法是编译生成，在本文 demo 中 render 方法是用户手写的`render: h =>h(aa)`，所以从`vm.render`开始不一样，那么从`vm.render`开始介绍组件如何渲染成最终的 DOM 。

## 一、vm._render

`vm._update(vm._render(), hydrating)`，在此处打个断点，按F11进入`vm._render()`方法中**。

```
Vue.prototype._render = function() {
    var vm = this;
    var ref = vm.$options;
    var render = ref.render;
    var _parentVnode = ref._parentVnode;
    vm.$vnode = _parentVnode;
    var vnode;
    try {
        currentRenderingInstance = vm;
        vnode = render.call(vm._renderProxy, vm.$createElement);
    } catch (e) {
        //...
    } finally {
        currentRenderingInstance = null;
    }
    return vnode;
}
```
此时的`vnode = render.call(vm._renderProxy, vm.$createElement)`中的`render`是用户手写的render方法`render: h =>h(aa)`，其中`h`是`vm.$createElement`。

`vm.$createElement`是在 Vue 初始化中通过`initRender(vm)`定义的。
```
function initRender(vm) {
    vm._c = function(a, b, c, d) {
        return createElement(vm, a, b, c, d, false);
    };
    vm.$createElement = function(a, b, c, d) {
        return createElement(vm, a, b, c, d, true);
    };
}
```
可以看到调用`vm.$createElement`实际上是调用`createElement(vm, a, b, c, d, true)`。

`vnode = render.call(vm._renderProxy, vm.$createElement)` **在此处打个断点，按三次F11进入`createElement`方法中**。

### 1、_createElement
```
var SIMPLE_NORMALIZE = 1;
var ALWAYS_NORMALIZE = 2;
function createElement(context, tag, data, children, normalizationType, alwaysNormalize) {
    if (Array.isArray(data) || isPrimitive(data)) {
        normalizationType = children;
        children = data;
        data = undefined;
    }
    if (isTrue(alwaysNormalize)) {
        normalizationType = ALWAYS_NORMALIZE;
    }
    return _createElement(context, tag, data, children, normalizationType)
}
```
要注意参数`alwaysNormalize`为`true`,故`normalizationType`有值为2。

最后调用`_createElement`,**此处打个断点，按F11进入`_createElement`方法中**。
```
function _createElement(context, tag, data, children, normalizationType) {
    if (normalizationType === ALWAYS_NORMALIZE) {
        children = normalizeChildren(children);
    } else if (normalizationType === SIMPLE_NORMALIZE) {
        children = simpleNormalizeChildren(children);
    }
    var vnode;
    if (typeof tag === 'string') {
        //...
    } else {
        vnode = createComponent(tag, data, context, children);
    }
    return vnode
}
```
从设定的场景中，从`render: h =>h(aa)`可以得知，参数`data`、参数`children`为 undefined ，故即是参数`normalizationType`为 2 等于`ALWAYS_NORMALIZE`，也可以忽略`children = normalizeChildren(children)`的逻辑过程。

参数`tag`为组件 aa 的选项对象。 
```
const aa ={
    template:'<div><p>{{aa}}<span>{{bb}}</span></p></div>',
    data(){
        return{
            aa:'欢迎',
            bb:'Vue'
        }
    }
}
```

故直接走 else 中的逻辑，调用`createComponent`生成组件类型的`vnode`。`vnode = createComponent(tag, data, context, children)`，**在此处打个断点，按F11进入`createComponent`方法中**。

### 2、createComponent
```
function createComponent(Ctor, data, context, children, tag) {
    var baseCtor = context.$options._base;
    if (isObject(Ctor)) {
        Ctor = baseCtor.extend(Ctor);
    }
    data = data || {};
    installComponentHooks(data);
    var name = Ctor.options.name || tag;
    var vnode = new VNode(
        ("vue-component-" + (Ctor.cid) + (name ? ("-" + name) : '')),
        data, undefined, undefined, undefined, context, {
            Ctor: Ctor,
            tag: tag,
            children: children
        }
    );
    return vnode
}
```
以上代码有三个关键步骤，构造子类构造函数，安装组件钩子函数和实例化 VNode 。

#### 1、构造子类构造函数baseCtor.extend
```
var baseCtor = context.$options._base;
if (isObject(Ctor)) {
    Ctor = baseCtor.extend(Ctor);
}
```
`_base`在`initGlobalAPI`函数中定义。
```
function initGlobalAPI(Vue) {
    Vue.options._base = Vue;
}
```
又在`this._init(options)`中，把`options`合并到`$options`上。
```
vm.$options = mergeOptions(
  	resolveConstructorOptions(vm.constructor),
  	options || {},
  	vm
);
```
所以`baseCtor`实际上就是 Vue 构造函数，再来看一下`Vue.extend`函数的定义。

```
var ASSET_TYPES = ['component','directive','filter'];
Vue.extend = function(extendOptions) {
    extendOptions = extendOptions || {};
    var Super = this;
    var SuperId = Super.cid;
    var cachedCtors = extendOptions._Ctor || (extendOptions._Ctor = {});
    if (cachedCtors[SuperId]) {
        return cachedCtors[SuperId]
    }

    var name = extendOptions.name || Super.options.name;
    if (name) {
        validateComponentName(name);
    }

    var Sub = function VueComponent(options) {
        this._init(options);
    };
    Sub.prototype = Object.create(Super.prototype);
    Sub.prototype.constructor = Sub;
    
    Sub.cid = cid++;
    Sub.options = mergeOptions(
        Super.options,
        extendOptions
    );
    Sub['super'] = Super;

    if (Sub.options.props) {
        initProps$1(Sub);
    }
    if (Sub.options.computed) {
        initComputed$1(Sub);
    }
    
    Sub.extend = Super.extend;
    Sub.mixin = Super.mixin;
    Sub.use = Super.use;
    
    ASSET_TYPES.forEach(function(type) {
        Sub[type] = Super[type];
    });
    
    if (name) {
        Sub.options.components[name] = Sub;
    }
    
    Sub.superOptions = Super.options;
    Sub.extendOptions = extendOptions;
    Sub.sealedOptions = extend({}, Sub.options);
    cachedCtors[SuperId] = Sub;
    return Sub
}
```
`Vue.extend`的作用就是创建一个 Vue 的子类 Sub。
```
var Sub = function VueComponent(options) {
    this._init(options);
};
Sub.prototype = Object.create(Super.prototype);
Sub.prototype.constructor = Sub;
```
这里采用原型链继承和借用构造函数继承的组合继承方式，创建一个继承于 Vue 的子类 Sub 并返回。

借用构造函数继承好理解，在子类 Sub 的构造函数`VueComponent(options)`中调用父类 Vue 的初始化方法`this._init(options)`。这样实例化子类 Sub 时就会执行`this._init(options)`，就再次走到 Vue 的初始化过程。

原型链继承为什么不采用`Sub.prototype = new Vue()`，因为这样做，有个缺点创建子类 Sub 实例时，要调用两次父类 Vue。

采用`Sub.prototype = Object.create(Super.prototype)`来实现原型链继承，不会再去调用父类 Vue。然后再把子类 Sub 的构造器constructor重新指向`Sub`。
```
var Super = this;
var SuperId = Super.cid;
var cachedCtors = extendOptions._Ctor || (extendOptions._Ctor = {});
if (cachedCtors[SuperId]) {
    return cachedCtors[SuperId]
}
Sub.cid = cid++;
cachedCtors[SuperId] = Sub;
return Sub
```
在创建过程中，对子类 Sub 做了缓存，避免多次执行`Vue.extend`时对同一个组件重复创建子类 Sub。

#### 2、安装组件钩子函数installComponentHooks
```
data = data || {};
installComponentHooks(data);
```
`installComponentHooks`**此处打个断点，按F11进入`installComponentHooks`方法中**。
```
var componentVNodeHooks = {
    init: function init(vnode, hydrating) {
        //...
    },
    prepatch: function prepatch(oldVnode, vnode) {
        //...
    },
    insert: function insert(vnode) {
        //...
    },
    destroy: function destroy(vnode) {
        //...
    }
};
var hooksToMerge = Object.keys(componentVNodeHooks);

function mergeHook(f1, f2) {
    var merged = function(a, b) {
        f1(a, b);
        f2(a, b);
    };
    merged._merged = true;
    return merged
}

function installComponentHooks(data) {
    var hooks = data.hook || (data.hook = {});
    for (var i = 0; i < hooksToMerge.length; i++) {
        var key = hooksToMerge[i];
        var existing = hooks[key];
        var toMerge = componentVNodeHooks[key];
        if (existing !== toMerge && !(existing && existing._merged)) {
            hooks[key] = existing ? mergeHook(toMerge, existing) : toMerge;
        }
    }
}
```
整个`installComponentHooks`的过程就是把`componentVNodeHooks`中的钩子函数合并到`data.hook`中，在合并过程中，如果某个钩子函数已经存在`data.hook`中，通过`mergeHook`方法做合并，在`mergeHook`方法中，返回一个依次执行这两个钩子函数的函数，即完成合并。

**此小节要记住`data.hook`存储了组件的钩子函数**。

#### 3、new VNode
```
var name = Ctor.options.name || tag;
var vnode = new VNode(
    ("vue-component-" + (Ctor.cid) + (name ? ("-" + name) : '')),
    data, undefined, undefined, undefined, context, {
        Ctor: Ctor,
        tag: tag,
        children: children
    }
);
return vnode
```
先来看一下 `VNode`类的构造函数，忽略掉跟设定的场景无关的代码。
```
var VNode = function VNode(tag, data, children, text, elm, context, componentOptions, asyncFactory) {
    this.tag = tag;
    this.data = data;
    this.children = children;
    this.text = text;
    this.elm = elm;
    this.context = context;
    this.key = data && data.key;
    this.componentOptions = componentOptions;
    this.componentInstance = undefined;
}
```
需要注意组件的`vnode`和普通元素节点的`vnode`不同，组件的`vnode`是没有`children`的。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0e99b7121bcf4b57be1760afbcedbb26~tplv-k3u1fbpfcp-zoom-1.image)

执行完`vm._render()`生成`vnode`，回到`vm._update`中，分析`vnode`如何生成真实 DOM 。

## 二、vm._update
```
Vue.prototype._update = function(vnode, hydrating) {
    var vm = this;
    var prevEl = vm.$el;
    var prevVnode = vm._vnode;
    var restoreActiveInstance = setActiveInstance(vm);
    vm._vnode = vnode;
    if (!prevVnode) {
        vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false);
    } else {
        vm.$el = vm.__patch__(prevVnode, vnode);
    }
    restoreActiveInstance();
}
```

执行`var prevVnode = vm._vnode`，`vm._vnode`是当前 Vue 实例生成的 Virtual DOM ，在设定的场景中是首次渲染，此时`vm._vnode`为 undefined ，故`prevVnode`为 undefined ，再执行`vm._vnode = vnode`，把当前 Vue 实例生成的 Virtual DOM 赋值给`vm._vnode`。

因为`prevVnode`为 undefined ，故执行`vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false)`.

在[上一篇文章](https://juejin.im/post/6871161430830235662#heading-0)中，介绍了`vm.__patch__`是如何定义的。

执行`vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false)`,最终是调用`patch`方法，**在此处打个断点，按F11进入`patch`方法中**。

### 1、patch
```
function patch(oldVnode, vnode, hydrating, removeOnly) {
    const insertedVnodeQueue = []
    if (isUndef(oldVnode)) {} else {
        const isRealElement = isDef(oldVnode.nodeType)
        if (!isRealElement && sameVnode(oldVnode, vnode)) {} else {
            if (isRealElement) {
                oldVnode = emptyNodeAt(oldVnode)
            }
            const oldElm = oldVnode.elm
            const parentElm = nodeOps.parentNode(oldElm)
            createElm(vnode, insertedVnodeQueue, parentElm, nodeOps.nextSibling(oldElm))
            if (isDef(parentElm)) {
                removeVnodes(parentElm, [oldVnode], 0, 0)
            } else if (isDef(oldVnode.tag)) {}
        }
    }
    return vnode.elm
}
```
- 参数`oldVnode`：上一次的 Virtual DOM ，设定的场景中的值为`vm.$el`，是`<div id="app"></div>` DOM 对象；
- 参数`vnode`：这一次的 Virtual DOM ；
- 参数`hydrating`：在非服务端渲染情况下为 `false`，可以忽略；
- 参数`removeOnly`： 是在`transition-group`场景下用，设定场景中没有，为`false`，可以忽略。

如果`oldVnode`不是 Virtual DOM 而是 DOM 对象，要把`oldVnode`用`emptyNodeAt`转成一个 Virtual DOM，并在其属性`elm`赋值上被转换的 DOM 对象，所以`oldElm`等同`vm.$el`，在用`nodeOps.parentNode(oldElm)`获取`oldElm`的父级 DOM 节点，此时为 body。
```
function emptyNodeAt (elm) {
    return new VNode(nodeOps.tagName(elm).toLowerCase(), {}, [], undefined, elm)
}
//nodeOps.parentNode
function parentNode (node) {
    return node.parentNode
}
```
执行`createElm(vnode, insertedVnodeQueue, parentElm, nodeOps.nextSibling(oldElm))`生成真实 DOM ，**在此处打个断点，按F11进入`createElm`方法中**,`nodeOps.nextSibling(oldElm)`取`oldElm`的下一个兄弟节点。

### 2、createElm
```
function createElm( vnode, insertedVnodeQueue, parentElm, refElm, nested, ownerArray, index) {
    if (createComponent(vnode, insertedVnodeQueue, parentElm, refElm)) {
        return
    }
}
```
- 参数`vnode`： Virtual DOM；
- 参数`insertedVnodeQueue`：钩子函数队列；
- 参数`parentElm`: 参数`vnode`对应 DOM 对象的父节点 DOM 对象；
- 参数`refElm`: 占位节点对象，例如，参数`vnode`对应 DOM 对象的下个兄弟节点；

在设定场景中，是要把组件渲染成 DOM ，会调用`createComponent`方法，**在此处打个断点，按F11进入`createComponent`方法中**。

### 3、createComponent
```
function createComponent(vnode, insertedVnodeQueue, parentElm, refElm) {
    var i = vnode.data;
    if (isDef(i)) {
        if (isDef(i = i.hook) && isDef(i = i.init)) {
            i(vnode, false);
        }
        if (isDef(vnode.componentInstance)) {
            initComponent(vnode, insertedVnodeQueue);
            insert(parentElm, vnode.elm, refElm);
            if (isTrue(isReactivated)) {
                reactivateComponent(vnode, insertedVnodeQueue, parentElm, refElm);
            }
            return true
        }
    }
}
```
执行完`var i = vnode.data;isDef(i = i.hook) && isDef(i = i.init)`，此时`i` 为`vnode.data.hook`中的`init`方法。`init`方法在`componentVNodeHooks`中定义，通过`installComponentHooks`方法将其合并到`data.hook`中。

`i(vnode, false)`，**在此处打个断点，按F11进入`init`方法中**。

此外还有注意到`insert(parentElm, vnode.elm, refElm)`这句代码，先提一下这句代码的作用是把组件内容生成的 DOM 插入父节点中，后面会详细介绍。

### 4、componentVNodeHooks.init
```
var componentVNodeHooks = {
    init: function init(vnode, hydrating) {
        if (vnode.componentInstance 
        	&&!vnode.componentInstance._isDestroyed 
            && vnode.data.keepAlive) {
            //...
        } else {
            var child = vnode.componentInstance  = 
            	createComponentInstanceForVnode( vnode, activeInstance);
            child.$mount(hydrating ? vnode.elm : undefined, hydrating);
        }
    },
}
```
`vnode.componentInstance`的含义是组件实例，此时组件实例还没创建，故`vnode.componentInstance`为 undefined，走 else 部分逻辑。通过`createComponentInstanceForVnode`方法创建一个 Vue 实例 `child`，调用`$mount`方法挂载组件。 

`createComponentInstanceForVnode( vnode, activeInstance)`**在此处打个断点，按F11进入`createComponentInstanceForVnode`方法**。

### 5、createComponentInstanceForVnode
```
function createComponentInstanceForVnode(vnode, parent) {
    var options = {
        _isComponent: true,
        _parentVnode: vnode,
        parent: parent
    };
    var inlineTemplate = vnode.data.inlineTemplate;
    if (isDef(inlineTemplate)) {
        options.render = inlineTemplate.render;
        options.staticRenderFns = inlineTemplate.staticRenderFns;
    }
    return new vnode.componentOptions.Ctor(options)
}
```
- 参数`vnode`: 要渲染的组件生成的 Virtual DOM。
- 参数`parent`: 要渲染的组件的父 Vue 实例，也就是上下文环境。

还记得在`createComponent`方法中，通过`Ctor = baseCtor.extend(Ctor)`创建了一个 Vue 的子类（组件）构造函数赋值给`Ctor`，然后在实例化 Vnode 中，通过 Vnode 构造函数的参数传递给`vnode`的属性`componentOptions`。所以`vnode.componentOptions.Ctor`就是要渲染组件的构造函数，`new`一下来生成组件实例。

`new vnode.componentOptions.Ctor(options)`，**在此处打个断点，按F11进入**。会发现走到`Vue.extend`方法中的
```
var Sub = function VueComponent (options) {
  	this._init(options);
}
```
执行`this._init(options)`，进行组件构造函数的初始化，又回到 Vue 构造函数的初始化，按F11进入`this._init`中，开始介绍组件内容的渲染过程

### 6、组件内容的渲染过程

#### 1、this._init
```
Vue.prototype._init = function(options) {
    var vm = this;

    if (options && options._isComponent) {
        initInternalComponent(vm, options);
    } else {
        //...
    }
    
    if (vm.$options.el) {
        vm.$mount(vm.$options.el);
    }
}
```

因为在`createComponentInstanceForVnode`方法中设置了`options = {_isComponent : true}`，
故`options._isComponent`为`true`，执行`initInternalComponent(vm, options)`来合并`options`，**在此次打个断点，进入`initInternalComponent`方法中**。

#### 2、initInternalComponent
```
function initInternalComponent(vm, options) {
    var opts = vm.$options = Object.create(vm.constructor.options);
    var parentVnode = options._parentVnode;
    opts.parent = options.parent;
    opts._parentVnode = parentVnode;

    var vnodeComponentOptions = parentVnode.componentOptions;
    opts.propsData = vnodeComponentOptions.propsData;
    opts._parentListeners = vnodeComponentOptions.listeners;
    opts._renderChildren = vnodeComponentOptions.children;
    opts._componentTag = vnodeComponentOptions.tag;

    if (options.render) {
        opts.render = options.render;
        opts.staticRenderFns = options.staticRenderFns;
    }
}
```
还记得用`Vue.extend`方法创建组件的构造函数时，有执行以下一段代码
```
Vue.extend = function(extendOptions){
    Sub.options = mergeOptions(
        Super.options,
        extendOptions
    );
}
```
在`createComponent`方法中执行`Ctor = baseCtor.extend(Ctor)`中调用`Vue.extend`，参数`Ctor`为 demo 中 aa 组件的选项对象，即`extendOptions`的值为
```
{
    template:'<div><p>{{aa}}<span>{{bb}}</span></p></div>',
    data(){
        return{
            aa:'欢迎',
            bb:'Vue'
        }
    }
}
```
经过`mergeOptions`方法合并，可以通过构造函数的属性`options`访问到 demo 中 aa 组件的选项对象。那么执行`var opts = vm.$options = Object.create(vm.constructor.options)`就可以用`vm.$options`获取到 demo 中 aa 组件的选项对象。

执行`opts.parent = options.parent;opts._parentVnode = parentVnode;`把之前通过`createComponentInstanceForVnode`函数传入的几个参数合并到`vm.$options`。

- `vm.$options.parentVnode`: 要渲染的组件生成的 Virtual DOM。
- `vm.$options.parent`: 要渲染的组件的父 Vue 实例，也就是上下文环境。

合并`options`完毕后回到`this._init`中执行`if (vm.$options.el){vm.$mount(vm.$options.el)}`，由于组件选项对象中没有`el`，在这里不执行`vm.$mount`挂载。

回到`componentVNodeHooks.init`钩子函数中，执行`child.$mount(hydrating ? vnode.elm : undefined, hydrating) `，这里不是服务端，故`hydrating`为`false`，相当于执行`child.$mount(undefined, false)`，对组件进行挂载，在此打个断点，按F11进入`child.$mount`方法中。

#### 3、child.$mount
```
var mount = Vue.prototype.$mount;
Vue.prototype.$mount = function(el, hydrating) {
    var options = this.$options;
    if (!options.render) {
        var template = options.template;
        if (template) {
            if (typeof template === 'string') {
                if (template.charAt(0) === '#') {
                    //...
                }
            } else if (template.nodeType) {
                template = template.innerHTML;
            } else {
                //...
            }
        } else if (el) {
            template = getOuterHTML(el);
        }
        if (template) {
            var ref = compileToFunctions(template, {
                outputSourceRange: "development" !== 'production',
                shouldDecodeNewlines: shouldDecodeNewlines,
                shouldDecodeNewlinesForHref: shouldDecodeNewlinesForHref,
                delimiters: options.delimiters,
                comments: options.comments
            }, this);
            var render = ref.render;
            var staticRenderFns = ref.staticRenderFns;
            options.render = render;
            options.staticRenderFns = staticRenderFns;
        }
    }
    return mount.call(this, el, hydrating)
};
```
在`initInternalComponent`方法中，把组件选项对象合并到`this.$options`，执行`var template = options.template`，
`template`为`<p>{{aa}}<span>{{bb}}</span></p>`。因`options.render`为 undefined ，故调用`compileToFunctions`方法把`template`转成成 render 方法，并挂载到`this.$options`上。

执行`return mount.call(this, el, hydrating)`，**在此处打个断点，按F11进入`mount`方法**。

```
Vue.prototype.$mount = function (el,hydrating) {
  el = el && inBrowser ? query(el) : undefined;
  return mountComponent(this, el, hydrating)
};
```
**这里要注意此时的`el`为 undefined**。执行`return mountComponent(this, el, hydrating)`，**在此处打个断点，按F11进入`mountComponent`方法**。

#### 4、mountComponent
```
function mountComponent(vm, el, hydrating) {
    vm.$el = el;
    var updateComponent;
    updateComponent = function() {
        vm._update(vm._render(), hydrating);
    };
    new Watcher(vm, updateComponent, noop, {
        before: function before() {
            if (vm._isMounted && !vm._isDestroyed) {
                callHook(vm, 'beforeUpdate');
            }
        }
    }, true);
    hydrating = false;

    if (vm.$vnode == null) {
        vm._isMounted = true;
        callHook(vm, 'mounted');
    }
    return vm
}
```
此时`vm`代表的是组件实例，不是Vue实例。另外此时`el`为 undefined，故`vm.$el`为 undefined，**这个要记住后面过程中会用到**。

实例化一个渲染Watcher，初始化的时候会执行回调函数，即执行`vm._update(vm._render(), hydrating)`，按F11进入`vm._render`方法。

#### 5、vm._render

```
Vue.prototype._render = function() {
    var vm = this;
    var ref = vm.$options;
    var render = ref.render;
    var _parentVnode = ref._parentVnode;
    vm.$vnode = _parentVnode;
    var vnode;
    vnode = render.call(vm._renderProxy, vm.$createElement);
    vnode.parent = _parentVnode;
    return vnode
};
```

`vm.$vnode`表示 Vue 实例的父 Virtual DOM，其值`vm.$options._parentVnode`是在执行`createComponentInstanceForVnode(vnode, parent)`时，内部有段代码`var options = { _parentVnode: vnode,}`，再通过`initInternalComponent`合并到`vm.$options`上，其中的`vnode`是当前要渲染的组件生成的 Virtual DOM，那么相对于组件的内容就是父 Virtual DOM，也可以叫作组件实例的父 Virtual DOM，如下图所示。
![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0e99b7121bcf4b57be1760afbcedbb26~tplv-k3u1fbpfcp-zoom-1.image)
此时的 render 方法，如下所示
```
(function anonymous() {
    with(this) {
        return _c('div', [_c('p', [_v(_s(aa)), _c('span', [_v(_s(bb))])])])
    }
})
```
执行`vnode = render.call(vm._renderProxy, vm.$createElement)`，把**组件的内容**生成`vnode`（Virtual DOM） 树，其过程可以看[上一篇文章](https://juejin.im/post/6871161430830235662#heading-16)。

执行`vnode.parent = _parentVnode`把组件内容的父 Virtual DOM，赋值给`vnode.parent`，最后返回`vnode`，如下图所示。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9893553778c2410392973d4ea92f92b7~tplv-k3u1fbpfcp-zoom-1.image)
按F11进入`vm._update`方法。

#### 6、vm._update
```
Vue.prototype._update = function(vnode, hydrating) {
    var vm = this;
    var prevEl = vm.$el;
    var prevVnode = vm._vnode;
    var restoreActiveInstance = setActiveInstance(vm);
    vm._vnode = vnode;
    if (!prevVnode) {
        vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false);
    } else {
        vm.$el = vm.__patch__(prevVnode, vnode);
    }
    restoreActiveInstance();
};
```

首次渲染，执行`vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false)`，**注意此时，传入`vm.__patch__`的参数`vm.$el`是 undefined **，最终是调用`patch`方法，按F11进入`patch`方法中。

#### 7、patch
```
function patch(oldVnode, vnode, hydrating, removeOnly) {
    var isInitialPatch = false;
    var insertedVnodeQueue = [];
    if (isUndef(oldVnode)) {
        isInitialPatch = true;
        createElm(vnode, insertedVnodeQueue);
    } else {
        
    }
    invokeInsertHook(vnode, insertedVnodeQueue, isInitialPatch);
    return vnode.elm
}
```
`oldVnode` 为 undefined，故执行`createElm(vnode, insertedVnodeQueue)`，**在此处打个断点，按F11进入`createElm`方法**。

#### 8、createElm
```
function createElm(vnode, insertedVnodeQueue, parentElm, refElm, nested, ownerArray, index) {
    var children = vnode.children;
    var tag = vnode.tag;
    if (isDef(tag)) {
        vnode.elm = nodeOps.createElement(tag, vnode);
        createChildren(vnode, children, insertedVnodeQueue);
        insert(parentElm, vnode.elm, refElm);
    } else if (isTrue(vnode.isComment)) {
    } else {
        vnode.elm = nodeOps.createTextNode(vnode.text);
        insert(parentElm, vnode.elm, refElm);
    }
}
```
`createElm`方法用来创建真实的 DOM 节点，并插入对应的父节点。详解介绍可以上看[上一篇](https://juejin.im/post/6871161430830235662/#heading-19)内容。

执行`createChildren(vnode, children, insertedVnodeQueue)`，按F11进入`createChildren`方法。

#### 9、createChildren
```
function createChildren(vnode, children, insertedVnodeQueue) {
    if (Array.isArray(children)) {
        checkDuplicateKeys(children);
        for (var i = 0; i < children.length; ++i) {
            createElm(children[i], insertedVnodeQueue, vnode.elm, null, true, children, i);
        }
    } else if (isPrimitive(vnode.text)) {
        nodeOps.appendChild(vnode.elm, nodeOps.createTextNode(String(vnode.text)));
    }
}
```
`createChildren`的逻辑很简单，实际上是遍历`vnode`子 Virtual DOM，递归调用`createElm`，这是一种常用的深度优先的遍历算法，在遍历过程中会把`vnode.elm`作为  的 `children[i]`（Virtual DOM）对应真实 DOM 的父节点传入。

当`children`不是数组时。判断`vnode.text`是否是基础类型，若是调用`nodeOps.createTextNode`生成一个纯文本节点，再调用`nodeOps.appendChild`插入到`vnode.elm`中。

递归调用`createElm`中如果当前已经没有子 Virtual DOM，执行`insert(parentElm, vnode.elm, refElm)`把生成的 DOM (`vnode.elm`) 插入到对应父节点（`parentElm`）中，因为是递归调用，子 Virtual DOM 会优先调用`insert`，所以整个 Virtual DOM 树生成真实 DOM 后的插入顺序是先子后父。
**在`insert(parentElm, vnode.elm, refElm)`处打个断点，按F11进入`insert`方法**。

#### 10、insert
```
function insert(parent, elm, ref$$1) {
    if (isDef(parent)) {
        if (isDef(ref$$1)) {
            if (nodeOps.parentNode(ref$$1) === parent) {
                nodeOps.insertBefore(parent, elm, ref$$1);
            }
        } else {
            nodeOps.appendChild(parent, elm);
        }
    }
}
```
- 参数`parent`：要插入节点的父节点
- 参数`elm`: 要插入节点
- 参数`ref$$1`：参考节点，会在参考节点前插入

`nodeOps.insertBefore`对应`insertBefore`方法，`nodeOps.appendChild`对应`appendChild`方法，
```
function insertBefore (parentNode, newNode, referenceNode) {
	parentNode.insertBefore(newNode, referenceNode);
}
function appendChild (node, child) {
	node.appendChild(child);
}
```
`insertBefore`方法和`appendChild`方法其实就是调用原生 DOM 的 API 进行 DOM 操作。

#### 11、回到createElm

等遍历完`vnode`所有子 Virtual DOM，执行`insert(parentElm, vnode.elm, refElm)`时，因为在`patch`中是这么调用`createElm`，执行`createElm(vnode, insertedVnodeQueue)`，只传递两个参数，故`parentElm` 为 undefined，那么生成真实的 DOM 树要怎么插到对应的父节点呢？

### 7、回到createComponent
```
function createComponent(vnode, insertedVnodeQueue, parentElm, refElm) {
    var i = vnode.data;
    if (isDef(i)) {
        var isReactivated = isDef(vnode.componentInstance) && i.keepAlive;
        if (isDef(i = i.hook) && isDef(i = i.init)) {
            i(vnode, false);
        }
        if (isDef(vnode.componentInstance)) {
            initComponent(vnode, insertedVnodeQueue);
            insert(parentElm, vnode.elm, refElm);
            if (isTrue(isReactivated)) {
                reactivateComponent(vnode, insertedVnodeQueue, parentElm, refElm);
            }
            return true
        }
    }
}
```

在执行`i(vnode, false)`中，执行了`var child = vnode.componentInstance = createComponentInstanceForVnode(vnode, activeInstance)`，所以`vnode.componentInstance`有值为当前组件实例。

执行`initComponent(vnode, insertedVnodeQueue)`，按F11进入`initComponent`方法。
```
function initComponent(vnode, insertedVnodeQueue) {
    vnode.elm = vnode.componentInstance.$el;
}
```
`vnode.componentInstance.$el`是在`vm._updat`方法中执行`vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false)`赋值的，`vm`为当前组件实例。

这里的`vnode`为组件生成的 Virtual DOM ，不是组件内容生成的 Virtual DOM 。

执行`insert(parentElm, vnode.elm, refElm)`，`parentElm`为组件的父节点，这里为 body，这样就把组件内容生成的 DOM 树插入到body中。

## 三、总结

对比一下[数据和模板（普通元素标签）渲染成 DOM](https://juejin.im/post/6871161430830235662) 的流程，在组件渲染成 DOM 的流程中，执行`vm.render`过程中，调用`createComponent`方法生成一个组件类型的`vnode`，在其中调用`Vue.extend`创建一个继承于 Vue 的组件构造函数。执行`vm.updata`过程中，调用`createElm`方法将这个 Virtual DOM 生产真实的 DOM ，在其中执行`createComponent`方法时执行组件的钩子函数`init`将组件构造函数实例化，重新走渲染成 DOM 的流程。如果组件的内容**都是**普通元素标签时，则走数据和模板渲染成 DOM 的流程，在生成真实的 DOM 树后要回到`createComponent`方法中调用`insert`方法插入到组件的父节点中渲染到页面上。如果组件的内容包含组件标签则又开始走组件渲染成 DOM 的流程，直到所有子组件的内容**都是**普通元素标签时，才会回到`createComponent`方法中调用`insert`方法插入到父组件的父节点中渲染到页面上。


