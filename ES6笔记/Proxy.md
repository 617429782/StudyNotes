## 元编程

特性:  
  生成代码 eval
  反射: 运行时修改语言结构
    自省: 在运行时访问程序内部属性, 如 Reflect、Object.keys、Array.forEach 等  
    自我修改: 代码可以修改自身, 如 push、pop  
    调解: 捕获、拦截行为, 如 Proxy、Object.defineProperty  
即: Reflect 和 Proxy 提供了元编程中 自省 和 调解 的能力 (虽然es6之前也有)

## Proxy 和 Reflect

Proxy 代理: 拦截并自定义 基本操作, 如: 属性查找、赋值、枚举、函数调用  
Reflect 反射: 每个 handler 都有一个同名的反射函数, 为其提供默认行为  
若需在 handler 中实现 默认行为, 推荐使用 Reflect 提供的接口, 因为其提供的是干净的底层能力  
以 getPrototypeOf 为例, Object 和 Reflect 都提供了该方法, 但 Object 接受非对象类型时会做类型转换, 而 Reflect 仅会抛出异常

基本用法: const p = new Proxy(target, handler)  
handler 支持 13 种拦截器, 可以通过 Reflect 的属性查看 https://cloud.tencent.com/developer/section/1192014

与 Object.defineProperty 不同的是, proxy 并不直接修改目标对象的 行为, 而是为其创建 代理器对象

```
  const o = {
    age: 10,
    say() {
      return 'my age is ' + this.age;
    }
  }
  const p = new Proxy(o, {
    // 属性查询
    get(target, prop, receiver) {
      console.log('Proxy.get:', receiver === p) // true
      return Reflect.get(target, prop)
    },
    // 赋值
    set(target, prop, value, receiver) {
      console.log('Proxy.set:', value)
      return Reflect.set(target, prop, value, receiver)
    },
    // delete 操作符的拦截器
    deleteProperty(target, prop) {
      console.log('Proxy.deleteProperty')
      return Reflect.deleteProperty(target, prop);
    },
    // in 操作符的拦截器
    has(target, prop) {
      console.log('Proxy.has')
      return Reflect.has(target, prop);
    },
    // Object.getOwnPropertyNames
    // Object.getOwnPropertySymbols
    // Object.keys / values
    // for in 循环
    ownKeys(target) {
      console.log('Proxy.ownKeys')
      return Reflect.ownKeys(target);
    },
    // 读取对象原型时 拦截, 如 Object.getPrototypeOf
    getPrototypeOf(target) {
      console.log('Proxy.getPrototypeOf')
      return Reflect.getPrototypeOf(target);
    },
    // 设置对象原型时 拦截, 如 Object.getPrototypeOf
    setPrototypeOf(target, proto) {
      console.log('Proxy.setPrototypeOf')
      return Reflect.setPrototypeOf(target, proto);
    },
    // 当 target 为 函数类型 时, 拦截其 调用、call、apply
    apply(target, ctx, args) {
      console.log('Proxy.apply')
      return Reflect.apply(target, ctx, args);
    },
    // 当 target 作为 构造函数 使用时, 触发该操作, 如 new proxy()
    construct(target, args) {
      console.log('Proxy.construct')
      return Reflect.construct(target, args);
    },
    // 定义代理对象某个属性时的属性描述时触发该操作, 如 Object.defineProperty
    defineProperty(target, prop, desc) {
      console.log('Proxy.defineProperty')
      Reflect.defineProperty(target, prop, desc)
    },
    // 获取代理对象某个属性的属性描述时触发该操作, 如
    // Object.getOwnPropertyDescriptor
    // Object.keys、Object.assign 等
    getOwnPropertyDescriptor(target, prop) {
      console.log('Proxy.getOwnPropertyDescriptor')
      return Reflect.getOwnPropertyDescriptor(target, prop);
    },
    // 判断一个代理对象是否是可扩展, 如 Object.isExtensible(proxy)
    isExtensible(target) {
      console.log('Proxy.isExtensible')
      return Reflect.isExtensible(target);
    },
    // 让一个代理对象不可扩展时触发该操作, 如 Object.preventExtensions(proxy)
    preventExtensions(target) {
      console.log('Proxy.preventExtensions')
      return Reflect.preventExtensions(target);
    },
  });
```

## Proxy.revocable 创建可销毁的代理器

```
  const o = { age: 10 }
  const { proxy, revoke } = Proxy.revocable(o, {
    get(target, prop, receiver) {
      console.log('Proxy.get:', receiver === proxy) // true
      return target[prop]
    },
  });
  proxy.age
  // Proxy.get: true
  // 10
  revoke() // 销毁代理器, proxy 变为销毁状态
  proxy.age // Error
```

## Proxy 解决了 Object.defineProperty 在 对象类型 表现上的痛点
 Object.defineProperty 作用在 数组 上时, 无法监听 数组成员 和 长度 的变化  
 Object.defineProperty 作用在 对象 上时, 无法监听 属性 的变化  
 vue 中的做法是劫持 push、pop 等会改变数组长度的原型方法, 加入触发依赖的逻辑, 但这依然没有解决直接修改 length 的情况  
 对于 数组成员 或 对象属性 的变化, vue 采用了递归劫持, 并提供了 $set 方法供开发者在必要的时候手动劫持  

## Proxy 的使用场景

(1) 数据校验: 修改 set  
(2) 功能扩展: 修改 get 以实现 数组 负索引 等  
(3) 模拟数组  
 数组的 length 难以模拟, 主要表现在:  
 对超出数组长度的下标赋值, length 会自动增加  
 修改 length 时, 数组的实际长度也会跟随变化  
 而这两个操作都可以触发 Proxy 的 set 拦截器, 因此模拟数组的关键在于判断 set 的触发是否来源于 修改 length  
 ps: 是否来源于 修改 length 这个条件并不准确, 正确的应该是: 是否可作为数组的索引

```
class Arr {
  constructor(length = 0) {
    this.length = length;
    for (let i = 0; i < length; i++) {
      Reflect.set(this, i, undefined);
    }
    return new Proxy(this, {
      set(target, prop, val) {
        const curLength = Reflect.get(target, 'length');
        if (prop !== 'length') {
          const index = Number(prop)
          if (index > curLength) {  // 对超出数组长度的下标赋值时, 修改length
            Reflect.set(target, 'length', index+1)
            for (let i = curLength; i < index; i++) {
              Reflect.set(target, i, undefined);  // 应该是 empty，但这无法手动赋值
            }
          }
        } else {  // 修改 length 时, 数组的实际长度也会跟随变化
          if (val < curLength) {
            for (let i = curLength - 1; i >= val; i--) {
              Reflect.deleteProperty(target, i);
            }
          } else if (val > curLength) {
            for (let i = curLength; i < val; i++) {
              Reflect.set(target, i, undefined);
            }
          }
        }
        return Reflect.set(target, prop, val);
      }
    })
  }
}
// test
var arr = new Arr(10); arr
arr.length = 20; arr
arr.length = 2; arr
arr[0] = 0; arr
arr[1] = 1; arr
arr[5] = 5; arr
```

(4) 响应式设计
