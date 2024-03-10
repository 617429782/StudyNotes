## let、const、var https://www.jianshu.com/p/0f49c88cf169
  1. V8的延迟编译，创建一个执行上下文时，会预先收集 var 变量 和 function 声明，存到 变量环境，这是变量提升的由来
  2. let、const 变量被存放于 环境记录器，会产生块级作用域，因为 暂时性死区 的存在，不能在声明前使用
  3. 块级作用域
    块级作用域 中的 let、const 变量 无法在 块作用域外访问
    块级作用域 中的 var 变量不受影响，function 为了兼容性的需要，由浏览器厂商以类似 var 的方式实现
  4. 变量提升
    1. var变量、函数声明 存在变量提升(先读后写不报错), let 和 const 不存在变量提示(先读后写会报错)
    2. 函数声明 -> 变量定义 -> 变量赋值, 函数的声明先于变量定义,且在变量提升阶段不会被更改为其他类型
      ```
        // 无论 var 和 function 谁先谁后，打印的都是 function 类型
        console.log(b); var b = 10; function b() {}; // f b(){}
        console.log(b); function b() {}; var b = 10; // f b(){}
        // 在 变量收集阶段 结束，执行阶段开始时后，类型限制不复存在
        var b = 10; function b() {}; console.log(b); // 10, 此时已经过了提升阶段进入赋值阶段
      ```
    3. 具名立即执行函数 属于函数表达式, 仅在函数内部有效, 不可修改
      ```
        var b = 10;
        (function b() {
          b = 20;
          console.log(b)  // f b() {...} 
        })()
      ```
    4. 变量提升会阻止内部变量晋升为全局变量
      ```
        (function () { b = 20; })(); console.log(b) // 20
        (function () { b = 20; var b = 10 })(); console.log(b) // 报错: not defined
      ```

## Symbol https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Symbol
  - Symbol 的特性与用途
  - 内置的 Symbol属性

## 函数的扩展功能
  默认参数, 给形参赋默认值 function(val = 0) {}
  剩余参数, function(...args) {}
  解构赋值, const [a, ...res] = [1,2,3]
  箭头函数, 没有自己的this(对应不能用call、apply、bind改变this、不能作为构造函数, 没有原型属性), 没有arguments(可以用剩余参数代替)

## 数组的扩展功能
  Array.of (解决了Array()传入单个数字时视为数组长度的怪异现象)
  Array.from 将类数组转为数组
  find / findIndex 检索
  fill 填充(会覆盖掉原有的值)

## 对象扩展功能
  Object.is 比较对象(不会进行类型转换, 结果基本与 === 相同, 并解决了 NaN !== NaN 和 +0 === -0 的问题)
  Object.assign 合并对象, 属于浅拷贝

## Promise
  1. Promise 特性
    1. 三个状态 pending 等待, fulfilled 已完成, rejected 已拒绝, 只能由 pending 向其他状态变化
      1. Promise.resolve      
      2. Promise.reject   
    2. 三个原型方法 then、catch、finally
      1. then(sCb, fCb): 注册完成态回调，返回一个 promise实例
      2. catch(): 注册拒绝态回调，返回一个 promise实例
      3. finally(): 所有实例都返回了状态, 无论是完成/拒绝
    3. 静态方法 all、race、any
      1. all([...]): 所有的实例 都返回了 完成状态
      2. race([...]): 返回最快得到结果的Promise实例, 无论状态
      3. any([...]): 新特性，出现第一个成功态时触发
  2. then/catch 调用特性
    1. 串行执行(链式调用): 上一个 Promise 状态改变后，下一个 Promise 才会执行   
      1. 上一个 then 的 return值 会作为 下一个 then 的 入参   
        ```
          new Promise((r) => r(1)).then(d => {
            console.log(d);            // 1
            return d + 1;              // 作为下一个then回调的参数
          }).then(d => console.log(d)) // 2

          // 其等价于
          const p1 = Promise.resolve(1);
          const p2 = p1.then(d => return d+1) // 这里的 d 是 p1 的内部变量
          p2.then(d => console.log(d))
        ```
      2. then 的 return值 为 promise 时，下一个 then/catch 的入参取决于 该 promise 的值
        ```
          new Promise((r) => r(1)).then(d => {
            console.log(d);            // 1
            return new Promise((r, j) => {
              r(10)                    // 作为下一个then回调的参数
            });              
          }).then(d => console.log(d)) // 10

          // 其等价于
          const p1 = Promise.resolve(1);
          const p2 = p1.then(d => console.log(d))
          const p3 = p2.then(d => new Promise(r => r(10)})
          p3.then(d => console.log(d))  // 因为 第二个then回调 是挂在 p3 上的，因此其执行与否取决于 p3 的状态，而不是 p2
        ```
    2. 并行执行: 相互执行不影响，即便是 嵌套执行
      ```
        const p = Promise.resolve(1);
        p.then(d => console.log(d+1)); // 2
        p.then(d => {
          console.log(d+2);            // 3

          p.then(d => {
            console.log(d+3);          // 4
          });
        });
      ```
  3. 手写 Promise 要点
    1. 发布订阅: 发布订阅模式，then/catch 注册回调，resolve/reject 时触发回调
    2. 延迟执行: 延迟 resolve/reject 到下个事件循环，以让 then() 在 resolve() 前生效
    3. 链式调用: 
      执行器: resolve / reject 接受参数，为 下一个 promise._value 赋值，同时修改当前 promise 的 state
      更新器: onFulfilled / onRejected 修改上一个 执行器 传下来的参数
      以链表的形式，每个promise在 callbacks 中持有下个 promise 的 更新器 和 执行器
      promise 在自身状态变化后调用 下一个 promise 的更新器，获取 处理后的数据，并作为下一个 promise 执行器的入参
    4. 边界处理: 
      1. 更新器 返回值是一个 promise
        此时 更新器的返回值 不能直接作为 下一个 执行器 的入参，需要先取得该 promise 已决后的 value
        要取得 promise 已决后的 value，最简单的方法就是 调用他的then，注册一个 临时监听器
        关键在于 临时监听器 拿到 已决后的 value，怎么传递给 后面的 监听器
        将 临时监听器 的 监听对象 称为 发布者，后面的 监听器，原先是在 发布者 的 callbacks 中存入了 自身的 执行器，供 发布者 已决后 调用
        那么，只需要在 临时监听器 状态变化时，执行 监听器注册在发布者中的 执行器即可
        那么，最简单的方法是什么呢，再执行一边 发布者 的执行器，他会自动调用订阅了自己的 订阅者的执行器
      2. 更新器 缺失
        链式调用中，某个 listener 没有通过 then 注册 更新器 时，直接调用 listener 的执行器

  ```
    type State = 'pending' | 'fulfilled' | 'rejected';

    interface Listener {
      onFulfilled: Function,
      onRejected: Function,
      resolve: Function,
      reject: Function,
    }

    class MPromise {
      then: Function;

      constructor(fn: Function) {
        let state: State = 'pending';
        let _value: any = null;
        const callbacks: Listener[] = []; // 收集监听当前Promise变化的 订阅者

        function exec() {
          // 延迟执行的目的是 让 then 先执行
          setTimeout(() => {
            callbacks.forEach(listener => {
              handle(listener);
            })
          }, 0)
        }
      
        function reject(value: any) {
          state = 'rejected';
          _value = value;
          exec();
        }
      
        // 链式调用的执行者, 其接受的值, 将作为 当前 Promise 的 最终value, 并传递给下一个 Promise
        function resolve(valueOrPromise: any) {
          if (valueOrPromise instanceof MPromise) {
            // 此时 更新器的返回值 不能直接作为 下一个 执行器 的入参, 需要先取得该 promise 已决后的 value
            // 要取得 promise 已决后的 value, 最简单的方法就是 调用他的then, 注册一个 临时监听器
            // 关键在于 临时监听器 拿到 已决后的 value, 怎么传递给 后面的 监听器
            // 将 临时监听器 的 监听对象 称为 发布者, 后面的 监听器, 原先是在 发布者 的 callbacks 中存入了 自身的 执行器, 供 发布者 已决后 调用
            // 那么, 只需要在 临时监听器 状态变化时, 执行 监听器注册在发布者中的 执行器即可
            // 那么, 最简单的方法是什么呢, 再执行一边 发布者 的执行器, 他会自动调用订阅了自己的 订阅者的执行器
            valueOrPromise.then(resolve, reject);
          } else {
            state = 'fulfilled';
            _value = valueOrPromise;
            exec();
          }
        }
      
        // 负责处理 订阅者
        // 若当前 promise 未决, 则将 订阅者 放入队列, 等待 状态变化
        // 当前 promise 状态变化后, 调用 订阅者 的 执行器, 将当前的 _value 传递过去
        function handle(listener: Listener) {
          // 当前状态 Promise 是 未决态, 则加进 当前 Promise 的回调池 中
          if (state === 'pending') {
            callbacks.push(listener);
          } else {
            // 当前状态 Promise 是 已决态, 则 先调用更新器 修改 传给下一个 promise 的值
            const updater = state === 'fulfilled' ? listener.onFulfilled : listener.onRejected;
            const executer  = state === 'fulfilled' ? listener.resolve : listener.reject;
            let nextPromiseValue = _value;
            if (updater) {
              // 更新器 的返回值有两种情况, 将决定下一个 执行器 的入参, 执行器(resolve) 中需要考虑到这种情况
              // 1. 非 promise, retrun 值直接作为下一个 resolve 的入参
              // 2. promise, 下一个 resolve 的入参 取已决态的 promise._value
              nextPromiseValue = updater(_value);
            }
            executer(nextPromiseValue);
          }
        }

        // 订阅, 将自身的 执行器 存入 订阅对象 的 callbacks
        this.then = function (onFulfilled: Function, onRejected: Function) {
          // onFulfilled / onRejected 更新器, 修改上一个 resolve 传下来的值
          // resolve / reject 执行器, 修改当前 promise.state, 并给下一个 promise._value 赋值
          return new MPromise((r: Function, j: Function) => {
            handle({
              onFulfilled,
              onRejected,
              resolve: r,
              reject: j,
            })
          });
        };

        fn(resolve, reject);
      }
    }

    // 测试用例 1: 串行
    new MPromise((resolve: Function) => { 
      setTimeout(() => {
        resolve(10);
      }, 500)
    }).then((d: any) => {
      console.log(d)
      return d + 1;
    }).then((d: any) => {
      console.log(d)
      return new MPromise((resolve: Function)  => {
        resolve(d + 1);
      });
    }).then((d: any) => {
      console.log(d)
    })

    // 测试用例 2: 并行、嵌套
    const p = new MPromise((resolve: Function) => { 
      setTimeout(() => {
        resolve(1);
      }, 500)
    })
    p.then((d: any) => {
      console.log('p1', d + 1)
      p.then((d: any) => console.log('p2', d + 2))
    })
    p.then((d: any) => console.log('p3', d + 3))
  ```

## generator / yield https://mp.weixin.qq.com/s/s3glJPuaIMU-uhFPFFCTBQ
  - 基本用法 (符号*声明迭代器，yield声明阶段结果，next方法步进)
    function* step() {
      yield 1;
      const temp = yield 2; // next 传入的参数会作为 上一个 yield 的返回值，之所以是上一个，是因为 赋值语句 右侧先执行 yield，此时程序被中断，没有完成赋值，等到下一次步进恢复
      console.log(temp)
      yield 3;
      return 'ending';
    }
    var m = step();
    m.next() // {value: 1, done: false}
    m.next() // {value: 2, done: false}
    m.next('this is param') // {value: 3, done: false}
    m.next() // {value: 'ending', done: true}

    // 生成器实例 m 是一个 Iterator 对象，其结构可以视为
    {
      next () {
        return { value: any, done: Boolean }
      }
    }
    // 当 m.done 为 true 后，m 的结构就变成了 
    { 
      next () { 
        return { value: undefind, done: true } 
      } 
    }

  - 特性
    1. 生成器函数执行时，不返回 return 值，而是返回 内部状态对象
    2. 分段执行，多次执行 next() 将得到不同阶段的结果
    3. 多个迭代器实例互不干扰

  - 注意点
    1. 不可用作构造函数
    2. 箭头函数不可用作生成器函数
    3. 

  - 手写
    function G(fn) {
      var 
      return {
        next() {
          value
        }
      }
    }

  - 答题要点
    1. generator / yield 是什么，解决了什么问题

## async / await
  1. 用法
    async 返回一个 fulfilled 状态的 Promise 实例
    await 等待 Promise 实例的异步返回

  2. async 与 promise 的执行顺序

## Set / Map / WeakSet / WeakMap
