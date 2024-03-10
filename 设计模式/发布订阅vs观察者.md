## 观察者模式

观察者模式 是 松耦合 的, 被观察者 维护一个 观察者 的列表, 在需要时通知 所有观察者, 因此观察者模式是 一对多 的  
但通常 观察者 也会维护自己观察过的 所有被观察者列表, 以便 新增/解除 观察, 因此 观察者 和 被观察者 也可以设计成 多对多 的  
典型的是 vue 的响应式原理  

```
// 被观察者类 伪代码
class Subject {
  obs: [],            // 观察该 Subject 的所有 Observer 实例
  addOb() {},
  removeOb() {},
  notify() {},        // 通知所有的 Observer 实例, 调用 update 方法
  ?depend() {},       // 非必须的, 用于将 自身 添加到 指定 Observer 实例的 subs 下
}

// 观察者类 伪代码
class Observer {
  ?subs: [],          // 非必须的, 观察过的所有 Subject 实例
  ?addSub() {},       // 同样是非必须的
  ?removeSub() {},    // 同样是非必须的
  update() {},
}

```
vue 的响应式设计中, 其 Dep 对应的是 Subject, Watcher 对应的是 Observer, 而其 Observer 类的作用是作为 响应式入口  
响应式设计的核心在于 让 创建Subject实例、添加Observer、通知Observer 三个步骤自动化  

```
  class Dep {
    constructor() {
      this.watchers = new Set();
    }
    addOb(sub) {
      this.watchers.add(sub);
    }
    removeOb(sub) {
      this.watchers.delete(sub)
    }
    depend() {
      // Dep.target 必须是 watcher 实例
      Dep.target && Dep.target.addDep(this)
    }
    notify() {
      this.watchers.forEach(watcher => {
        typeof watcher?.update === 'function' && watcher.update()
      })
    }
  }

  class Watcher {
    constructor(getter, cb) {
      this.deps = new Set();
      // 在vue中, getter支持 函数 和 属性链取值 两种方式, 属性链也会被转成函数的形式
      this.getter = getter;
      this.cb = cb;
      this.get()
    }
    addDep(dep) {
      dep.addOb(this)
    }
    clearDep() {   // 在vue中, watcher通常不是一对一地解除观察, 而是在销毁时统一清除
      for (const dep of this.deps) {
        this.deps.delete(dep);
        dep.removeOb(this);
      }
    }
    get() {
      Dep.target = this;
      // 通过调用 getter 来触发 target 的 get, 完成 依赖搜集
      let value = this.getter.call(null)   // vue 中, getter 的上下文是 组件
      Dep.target = null;
      return value;
    }
    update() {
      const value = this.get()
      const oldValue = this.value
      this.value = value
      typeof this?.cb === 'function' && this.cb.call(null, value, oldValue) // vue 中, cb 的上下文是 组件
    }
  }

  function defineReactive(target, key) {
    let _value = target[key];
    const dep = new Dep();  // 创建Subject实例 的自动化
    Object.defineProperty(target, key, {
      get() {
        dep.depend();       // 添加Observer 的自动化
        return _value;
      },
      set(value) {
        _value = value;
        dep.notify();       // 通知Observer 的自动化
      }
    })
  }

  const o = { age: 1 }
  defineReactive(o, 'age')
  new Watcher(() => {     // vue 中创建 Watcher 实例的时机有, mountComponent、$watch、initComputed
    return o.age
  }, (val, old) => {
    console.log(val, old)
  })
  o.age = 123
```

## 发布订阅

发布订阅模式 是 无耦合 的, 发布者 不直接通知 订阅者, 两者互不相识
发布者 与 订阅者 通过第三方来交互, 发布订阅关系表 维护在第三方
典型的是 事件机制

