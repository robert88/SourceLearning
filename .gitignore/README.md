# 看一下从 new Vue()开始到页面看到真实dom都经历了什么？

vue版本：2.5.2
main.js
```javascript
import Vue from 'vue'
import App from './App'
Vue.config.productionTip = false
/* eslint-disable no-new */
new Vue({
  el: "#app",
  render: h => h(App)
});
```

## 分析
在new Vue之前也就是会做一些初始化的工作,列举部分重要的代码

```javascript
/*
initMixin给Vue.prototype添加：
_init函数,
...
*/
initMixin(Vue);
/*
stateMixin给Vue.prototype添加：
$data属性,
$props属性,
$set函数,
$delete函数,
$watch函数,
...
*/
stateMixin(Vue);
/*
eventsMixin给Vue.prototype添加：
$on函数,
$once函数,
$off函数,
$emit函数,
$watch方法,
...
*/
eventsMixin(Vue);
/*
lifecycleMixin给Vue.prototype添加:
_update方法:私有方法,用于更新dom,其中调用_patch产生跟新后的dom,
$forceUpdate函数,
$destroy函数,
...
*/
lifecycleMixin(Vue);
/*
renderMixin给Vue.prototype添加:
$nextTick函数,
_render函数,
...
*/
renderMixin(Vue);
```

至此，部分常见的我们经常看到的初始化工作已经做完，现在我们知道我们在vm上用的$开头的方法都是在一加载vue.js完成后就挂在了Vue.prototype,我们的vm实例就可以用这些方法了。
下面当我们正式开始new Vue()

```javascript
//调用this._init()
new Vue(options);
/*
this._init函数
依次调用了跟vm相关的初始化函数:
initLifecycle:给vm挂在一下属性
  $parent:undefined,
  $root:vm,
  $children:[],
  $refs:{},
  _isMounted:false,
  _isDestoryed:false,
  _watcher:null,
  ...
initEvents:初始化事件.

initRender:主要做了一下事情:
  给vm添加_c函数和$createElement，其实是createElement别名， 
  给vm添加$attrs和$listeners属性，$attrs & $listeners are exposed for easier HOC creation

callHook(vm, 'beforeCreate')：此时我们看到触发了beforeCreate钩子，此时的vm有哪些属性应该一目了然了.

initInjections(vm)： resolve injections before data/props，我们可以看到在初始化inject时还没有data和props

initState:主要进行初始化:
  data：initData(vm),
  props:initProps(vm,opt.props)
  computed:initComputed(vm,opt.computed),
  methods:initMethods(vm,opt.methods),
  watch:initWatch(vm,opt.watch)

initProvide:resolve provide,根据源码注释我们可以知道，在provide中可以使用props和data

callHook(vm, 'created'):此时生命周期created触发，我们能访问data,prop,provide等等
下面最后一步至关重要：
 if (vm.$options.el) {
      vm.$mount(vm.$options.el);
    }
这里是判断我们是否传入了el，属性，传入了则调用$mount方法挂载内容到el所在节点下
*/
 this._init(options)
//在执行完 this._init() 后进入了最最重要的一步，挂载组件
//程序接着往下走回执行:mountComponent
/*
  函数 mountComponent 主要做了下面这些事情:
  触发 beforeMount() 钩子函数,
  声明 updateComponent 函数，里面调用了 vm_update(vm._render(),...)，这两个方法作用下面执行到的时候说一下.
  new Watcher(),此时会传入updateComponent函数,并随后执行此函数，执行后会发生一些函数执行，我只列举比较重要的大流程函数：
  Vue._render：执行由vue-loader生成的render函数或者自己写的render函数，最终返回一个由createComponent(非createPatchFunction内部的)产生的vnode.
  createComponent(非createPatchFunction内部):创建组件虚拟节点，此函数返回一个vnode，表示vue组件的vnode.
  vm._update:接收上面的vnode参数，这里面会触发VM.__patch__函数，这个函数里面最终返回的结果就是我们在html页面写的空的div,但是里面有了真实的内容，此时页面可以看到内容了，
  触发 mount() 钩子函数，这个mount钩子每个组件实例会在自己的insert hook中调用

*/
mountComponent()
```

到了这里，函数moutComponent执行完毕并且返回了vm,
等 this._init()执行完我们在页面就可以看到内容了

## 总结

我们可以大致总结一下从new Vue开始都大致执行了哪些重要的方法

```javascript
new Vue();
Vue.prototype._init();
Vue.prototype.$mount();
mountComponent();
new Watcher();
Watcher.prototype.get();
updateComponent();
Vue.prototype._render();
render();
createElement()
Vue.prototype._update();//这里面会执行vm.$el=vm.__patch__(),最终根vm的$el就有了真实dom值
Vue.prototype.__patch__();//这个应该是最重要的方法了，他返回了真实的dom节点。
```

在 vm.__patch__中会产生一个属于App.vue这个虚拟节点的实例,然后再次调用该实例的_init()方法，然后依次执行步骤2到12,继而完成app组件的挂载,最终new Vue出来的vm的$el,就是所有的真实dom。
