# 手写Vue2.0源码（九）-侦听属性原理

此篇主要手写 Vue2.0 源码-**侦听属性**

上一篇咱们主要介绍了 Vue [组件原理](https://juejin.cn/post/6954173708344770591) 深入了解了 Vue 组件化开发的特色 此篇将介绍我们日常业务开发使用非常多的侦听属性的原理

------

### 正文

```html
<script>
  // Vue实例化
  let vm = new Vue({
    el: "#app",
    data() {
      return {
        aa: 1,
        bb: 2,
      };
    },
    template: `<div id="a">hello 这是我自己写的Vue{{name}}</div>`,
    methods: {
      doSomething() {},
    },
    watch: {
      aa(newVal, oldVal) {
        console.log(newVal);
      },
      // aa: {
      //   handle(newVal， oldVal) {
      //     console.log(newVal);
      //   },
      //   deep: true
      // },
      // aa: 'doSomething',
      // aa: [{
      //   handle(newVal， oldVal) {
      //     console.log(newVal);
      //   },
      //   deep: true
      // }]
    },
  });
  setTimeout(() => {
    vm.aa = 1111;
  }, 1000);
</script>
```

侦听属性的写法很多 可以写成 字符串 函数 数组 以及对象 对于对象的写法自己可以增加一些 options 用来增强功能 侦听属性的特点是监听的值发生了变化之后可以执行用户传入的自定义方法

#### 1.侦听属性的初始化

```javascript
javascript复制代码// src/state.js

// 统一初始化数据的方法
export function initState(vm) {
  // 获取传入的数据对象
  const opts = vm.$options;
  if (opts.watch) {
    //侦听属性初始化
    initWatch(vm);
  }
}

// 初始化watch
function initWatch(vm) {
  let watch = vm.$options.watch;
  for (let k in watch) {
    const handler = watch[k]; //用户自定义watch的写法可能是数组 对象 函数 字符串
    if (Array.isArray(handler)) {
      // 如果是数组就遍历进行创建
      handler.forEach((handle) => {
        createWatcher(vm, k, handle);
      });
    } else {
      createWatcher(vm, k, handler);
    }
  }
}
// 创建watcher的核心
function createWatcher(vm, exprOrFn, handler, options = {}) {
  if (typeof handler === "object") {
    options = handler; //保存用户传入的对象
    handler = handler.handler; //这个代表真正用户传入的函数
  }
  if (typeof handler === "string") {
    //   代表传入的是定义好的methods方法
    handler = vm[handler];
  }
  //   调用vm.$watch创建用户watcher
  return vm.$watch(exprOrFn, handler, options);
}
```

initWatch初始化Watch对数组进行处理 createWatcher处理Watch的兼容性写法 包含字符串 函数 数组 以及对象 最后调用$watch 传入处理好的参数进行创建用户Watcher

#### 2.$watch

```javascript
javascript复制代码//  src/state.js
import Watcher from "./observer/watcher";
Vue.prototype.$watch = function (exprOrFn, cb, options) {
  const vm = this;
  //  user: true 这里表示是一个用户watcher
  let watcher = new Watcher(vm, exprOrFn, cb, { ...options, user: true });
  // 如果有immediate属性 代表需要立即执行回调
  if (options.immediate) {
    cb(); //如果立刻执行
  }
};
```

原型方法$watch 就是创建自定义 watch 的核心方法 把用户定义的 options 和 user:true 传给构造函数 Watcher

#### 3.Watcher 改造

```javascript
javascript复制代码// src/observer/watcher.js

import { isObject } from "../util/index";
export default class Watcher {
  constructor(vm, exprOrFn, cb, options) {
    // this.vm = vm;
    // this.exprOrFn = exprOrFn;
    // this.cb = cb; //回调函数 比如在watcher更新之前可以执行beforeUpdate方法
    // this.options = options; //额外的选项 true代表渲染watcher
    // this.id = id++; // watcher的唯一标识
    // this.deps = []; //存放dep的容器
    // this.depsId = new Set(); //用来去重dep

    this.user = options.user; //标识用户watcher

    // 如果表达式是一个函数
    if (typeof exprOrFn === "function") {
      this.getter = exprOrFn;
    } else {
      this.getter = function () {
        //用户watcher传过来的可能是一个字符串   类似a.a.a.a.b
        let path = exprOrFn.split(".");
        let obj = vm;
        for (let i = 0; i < path.length; i++) {
          obj = obj[path[i]]; //vm.a.a.a.a.b
        }
        return obj;
      };
    }
    // 实例化就进行一次取值操作 进行依赖收集过程
    this.value = this.get();
  }
  //   get() {
  //     pushTarget(this); // 在调用方法之前先把当前watcher实例推到全局Dep.target上
  //     const res = this.getter.call(this.vm); //如果watcher是渲染watcher 那么就相当于执行  vm._update(vm._render()) 这个方法在render函数执行的时候会取值 从而实现依赖收集
  //     popTarget(); // 在调用方法之后把当前watcher实例从全局Dep.target移除
  //     return res;
  //   }
  //   把dep放到deps里面 同时保证同一个dep只被保存到watcher一次  同样的  同一个watcher也只会保存在dep一次
  //   addDep(dep) {
  //     let id = dep.id;
  //     if (!this.depsId.has(id)) {
  //       this.depsId.add(id);
  //       this.deps.push(dep);
  //       //   直接调用dep的addSub方法  把自己--watcher实例添加到dep的subs容器里面
  //       dep.addSub(this);
  //     }
  //   }
  //   这里简单的就执行以下get方法  之后涉及到计算属性就不一样了
  //   update() {
  //     // 计算属性依赖的值发生变化 只需要把dirty置为true  下次访问到了重新计算
  //     if (this.lazy) {
  //       this.dirty = true;
  //     }else{
  //       // 每次watcher进行更新的时候  可以让他们先缓存起来  之后再一起调用
  //       // 异步队列机制
  //       queueWatcher(this);
  //     }
  //   }
  //   depend(){
  //     // 计算属性的watcher存储了依赖项的dep
  //     let i=this.deps.length
  //     while(i--){
  //       this.deps[i].depend() //调用依赖项的dep去收集渲染watcher
  //     }
  //   }
  run() {
    const newVal = this.get(); //新值
    const oldVal = this.value; //老值
    this.value = newVal; //现在的新值将成为下一次变化的老值
    if (this.user) {
      // 如果两次的值不相同  或者值是引用类型 因为引用类型新老值是相等的 他们是指向同一引用地址
      if (newVal !== oldVal || isObject(newVal)) {
        this.cb.call(this.vm, newVal, oldVal);
      }
    } else {
      // 渲染watcher
      this.cb.call(this.vm);
    }
  }
}
```

咱们主要关注非注释的地方 这里主要改造有两点

1.实例化的时候为了兼容用户 watch 的写法 会将传入的字符串写法转成 Vue 实例对应的值 并且调用 get 方法获取并保存一次旧值

2.run 方法判断如果是用户 watch 那么执行用户传入的回调函数 cb 并且把新值和旧值作为参数传入进去

#### 4.侦听属性的思维导图

![Vue2.0源码-侦听属性.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/61fa92688a1e4ed2b000d362939a1133~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

## 小结