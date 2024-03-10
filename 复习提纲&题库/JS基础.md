## JS类型
  1. 基本类型 String, Boolean, Number, Null, Undefined, Symbol
    基本类型 变量值 存储在 栈空间(基于作用域栈), 大小固定, 寻址速度快
    基本类型 变量声明后不可改变, 修改时将开辟一块新的内存空间替换旧空间, 旧空间将被垃圾回收

  2. 引用类型 Object
    引用类型 变量值 存放于堆空间(其指针地址存于栈空间), 可动态改变大小

## 执行上下文 / 闭包 / 作用域
  1. 执行上下文(this、词法环境(作用域)、变量环境): JS引擎执行一段代码所需的全部信息
    绑定this: 被引用对象调用时, 指向调用者, 否则指向 window (严格模式下指向 undefined)
    创建词法环境(作用域): 
      环境记录器: 用于存储 变量/函数 的 key-value 映射, 即 存储 函数内部的变量(let const) 和 arguments
      外部环境(父级作用域): 父级作用域的引用, 即 函数外部的变量
    创建变量环境: 也是词法环境的一种, 用于存储 var 变量、function 声明
  2. 执行栈: js引擎执行代码时首先创建全局上下文, 执行过程中 遇到函数调用 则为其 创建函数上下文 并入栈, 执行完则出栈
  3. 作用域链
    1. 作用域是执行上下文的一部分, 执行上下文创建词法环境时会 创建用于存储函数内部变量的环境记录器, 并持有父级环境的引用
    2. JS引擎在读取变量时首先在当前作用域查找, 没有找到则逐层到父级环境查找, 这是其链式的特性
  4. 闭包: 带有执行环境的函数
    1. JS引擎执行代码时首先会创建全局执行上下文, 遇到函数调用则创建函数上下文并入栈, 执行完毕后出栈, 并在垃圾回收中释放作用域中变量的引用
    2. 若在函数执行完后, 仍有其他地方持有了该函数中定义的变量, 则无法在垃圾回收中销毁, 这是闭包的典型应用: 函数在调用时访问声明环境的作用域
  5. 变量提升
    1. var 变量会在执行上下文的开头被初始化为 undefined, 因此可以在声明前使用而不报错, 称为变量提升  
    2. 其源于创建执行上下文对 var、let 等不同的处理  
    3. let、const 被初始化为 uninit 状态, function 也被初始化为 func 类型, 存储在 词法环境 中  
    4. 而 var 被初始化为 undefined, 存储在 变量环境 中  

## this / bind / call / apply
  1. this 绑定的五种规则:  
    默认绑定: 函数没有调用者时, this 指向 window
    隐式绑定: 函数有调用者时, this 指向 调用者
    显示绑定: 函数以 call、apply 的方式调用时, this 指向第一个参数, bind 方法会返回一个具有固定 this 的新函数
    new绑定: 构造函数被调用时, this指向生成的对象
    箭头函数绑定: 为箭头函数创建执行上下文时, 不会绑定this, 其this指向父级执行上下文的this

  2. 手写 new
    分析: 创建新对象、以新对象作为this调用构造函数、将新对象的 __proto__ 指向函数的原型、返回这个新对象
    function myNew (fn, ...args) {
      const o = Object.create(null)
      o.__proto__ = fn.prototype
      fn.apply(o, args)
      return o
    }

  3. 手写 call / apply / bind
    // 利用 隐式绑定, 让函数的 this 指向调用者
    Function.prototype.myCall = function(target, ...args) {
      target.fn = this  // 此处的this即函数对象本身
      target.fn(...args);
      delete target.fn
    }

    Function.prototype.myApply = function(target, args) {
      target.fn = this  // 此处的this即函数对象本身
      target.fn(...args);
      delete target.fn
    }

    // 利用闭包, 让返回的函数在执行时仍然可以访问 定义时的词法环境
    Function.prototype.myBind = function(target, ...args) {
      const self = target;
      const fn = this;
      return function () {
        self.fn = fn  // 此处的this即函数对象本身
        self.fn(...args);
        delete self.fn
      }
    }

## 原型 / 继承
  1. 原型链 https://www.jianshu.com/p/7119f0ab67c0
    prototype 是 函数的原型对象, 它会被函数生成实例的 __proto__ 引用
    函数原型对象中的属性和方法是所有实例共享的
    所有构造函数的原型链最终都将指向 Object.prototype, Object.prototype.__proto__ === null

  2. es5 继承的实现方式
    1. 原型链继承: 子类的原型 指向 父类的实例
      ```
        Son.prototype = new Parent()
      ```
      可以继承父类的构造属性和原型属性
      缺点是不能向父类传参
    
    2. 构造函数继承 在子类中实例化父类
      ```
        Son(...args) {
          Parent.call(this, ...args)
        }
      ```
      可以向父类传参, 继承父类的构造属性
      无法继承父类的原型属性(因为不是 new 出来的父实例)

    3. 原型构造组合继承 子类的原型对象指向父类, 以继承父类的构造/原型属性, 然后在子类实例化时, 通过call参数调用父类构造函数, 以覆盖之前继承的父类构造属性
      function Son(...args) {
        Parent.call(this, ...args) 
      }; 
      Son.prototype = new Parent()  
    
    原型式继承
    寄生式继承

## 深浅拷贝
  1. JSON.stringify 方法的缺陷
    忽略 undefined、Symbol
    函数、Date、正则对象无法正确序列化
    循环引用报错
  2. 手写深拷贝: 递归、循环引用
    ```
      function deepClone(source, cache = []) {  // cache 可以改用 weakmap
        // 基本类型
        if (source === null || typeof source !== 'object') return source;

        // 用哈希表缓存已出现的属性, 解决循环引用
        const hit = cache.find(v => v === source)
        if (hit) return hit;
        cache.push(source);

        // 递归处理
        const result = Array.isArray(source) ? [] : {};
        Object.keys(source).forEach(k => { // 如果是for in循环择要考虑原型属性, 用hasOwnProperty判断
          result[k] = deepClone(source[k], cache);
        })
        return result
      }
    ```

## 事件机制
  1. 事件机制
    1. 冒泡、捕获: 鼠标点击时, 本身没有坐标信息, 由操作系统计算并提供给浏览器
    2. 浏览器根据坐标自上而下找到对应元素的过程称为捕获
    3. 冒泡则是人类处理事件的逻辑 (按了遥控器按钮也就按了遥控器)
  2. addEventListener (eventType, cb, [Boolean｜Object])
    1. 第三个参数 控制 回调的触发时机 在 捕获 还是 冒泡(false, 默认) 阶段, 也可以传递对象进行详细控制 { once, passive(承诺不会进行preventDefault), useCapture }
  3. 事件委托: 过多的子元素事件绑定会造成无谓的内存浪费, 可以将事件绑定在父元素上, 通过 e.target 来判断触发事件的元素

## 事件循环 https://mp.weixin.qq.com/s/VZwAZcmAJGWeWrqRbiveaw
  1. 简介
    1. 事件循环是为了解决JS单线程的任务阻塞问题
    2. JS脚本执行过程中产生的异步事件, 如setTimeout、ajax等, 将由浏览器辅助线程添加到 任务队列, 等待下一次循环执行
    3. 辅助线程将任务添加进队列后并不通知主线程, 因此需要主线称以类似while循环的方式查看 任务队列 中是否有待执行任务
    4. ES6之后JS拥有了异步的能力, 即Promise, 其产生的异步任务被称为 微任务, 被添加到 微任务队列, 在本次 宏任务清空调用栈 后执行

  2. 常见的宏任务和微任务有哪些
    1. 宏任务: 主代码块 / setTimeout / setInterval / requestAnimationFrame / setImmediate(仅IE) / MessageChannel
    2. 微任务: promise.then/catch/finally, MutationObserver(监听DOM结构变化的接口), node中还有 process.nextTick

  3. node 中的事件循环跟浏览器中有什么区别？
    1. 浏览器中 微任务队列 在每个 宏任务 执行完后清空并执行
    2. node 中, 微任务执行时机穿插在 单次循环的各个阶段, 在 11版本以后, 表现才与浏览器一致
      ```
        setTimeout(() => {
          console.log('t1')
          Promise.resolve().then(function() {
            console.log('p1')
          })
        }, 0)
        setTimeout(() => {
          console.log('t2')
          Promise.resolve().then(function() {
            console.log('p2')
          })
        }, 0)

        // 浏览器 & node11以后版本: t1 -> p1 -> t2 -> p2
        // node11之前版本: t1 -> t2 -> p1 -> p2
      ```
    3. process.nextTick 回调在 promise.then 之前
    4. setImmediate 回调执行在 check 阶段, 开发者需要关注的有四个阶段
      (1) timer: 执行 setTimeout / setInterval 回调
      (2) poll: 执行I/O回调, 当I/O队列为空后将会进入 check 阶段
      (3) check: 执行 setImmediate回调
      (4) close: 执行关闭回调, 如 socket.destroy
      因此 setImmediate 与 setTimeout 的顺序取决于 setTimeout 被安排在哪次事件循环中, 谁先谁后都有可能
      但是如果在I/O回调中, setImmediate一定先执行, setTimeout将被排在下一次事件循环
      如果想 setTimeout优先, 那么只要让事件循环开始的事件推迟, 即执行一段同步任务, 甚至是微任务, 确保 setImmediate 执行前 timer 已经进入循环

## 防抖 / 节流  
  1. 接受2个参数, 分别是 执行回调 和 间隔时间, 返回包装后的函数
    ```
      function lazy (cb, delay) {
        return (...args) => { cb.apply(null, args) } 
      }
    ```
  2. 节流: 包装函数被多次执行时, 判断 调用间隔 是否大于 间隔时间
    ```
      function lazy (cb, delay) {
        let last = Date.now();
        return (...args) => {
          const now = Date.now()
          if ((now - last) > delay) {
            last = now;
            cb.apply(null, args)
          }
        } 
      }
    ```
  3. 防抖: 取消短时间内的重复执行, 
    ```
      function lazy (cb, delay) {
        let timer;
        return (...args) => {
          if (timer) clearTimeout(timer);
          timer = setTimeout(() => {
            cb.apply(null, args) 
          }, delay)
        } 
      }
    ```
  4. 节流 + 防抖: 在保证执行的前提下, 减少执行总数
    ```
      function lazy (cb, delay) {
        let last = Date.now();
        let timer;
        return (...args) => {
          const now = Date.now()
          let done = false;
          // 节流
          if ((now - last) > delay) {
            last = now;
            done = true;
            cb.apply(null, args)
          }
          // 防抖
          if (!done) {
            if (timer) clearTimeout(timer);
            timer = setTimeout(() => {
              cb.apply(null, args) 
            }, delay)
          }
        }
      }

      window.addEventListener('scroll', lazy(()=>{console.log(1)},500))
    ```

## 函数式编程 https://zhuanlan.zhihu.com/p/363757919
  1. 纯函数: 函数式编程的本质
    无状态: 在任意上下文中调用任意次数, 返回的结果都是一样的
    无耦合: 返回值只取决于入参, 不依赖/修改外部变量, 也应该避免修改引用类型的入参
  2. 优点: 易调试、易组合
  3. 缺点: 不适用于需要保持状态的场景, 比如分页查询, 更多得用于 工具库
  4. 应用场景
    1. 函数合成: const entry = compose(add, minus, ...) 
      ```
        function compose (init, ...fns) {
          let result = init;
          fns.forEach(f => result = f(result))
          return result
        }

        // 测试用例
        compose(1, value => {
          return value + 10;
        }, value => {
          return value - 1;
        })
      ```
    2. 函数柯理化: sum(1)(2)(3)
      柯理化特点: 将 函数 作为参数 或 返回值
      思路: 递归, 需要额外的结束信号
      ```
        function sum(init = 0) {
          return next => {
            // 将传入 undefined 作为递归结束的信号, 
            return next === undefined ? init : sum(init + next)
          }
        }
        sum(1)(2)(3)()
      ```

## worker
  [js高级程序设计4.md](../读书笔记/js高级程序设计4.md)

## 模块化 https://zhuanlan.zhihu.com/p/299658985
  1. CommonJS (node): require + module.exports
    1. 特性 运行时加载 (动态加载, 模块路径可以动态生成)
      1. 同步加载: 先执行 依赖模块 获得 export 对象, 再执行 当前模块
      2. 缓存特性: 被加载过的依赖 会缓存 export 对象, 再次被加载 直接返回
      3. 输出值拷贝: export对象的属性在导出的那一刻确定, 后续依赖模块中的修改不影响当前模块
    2. require 的查找规则 (node)
      1. node 核心模块: 如 fs、path、http, 其在node进程启动时就以二进制的形式被加载到内存, 直接返回
      2. 相对路径: 先尝试补全后缀(js/json/node), 以文件为目标定位, 找不到则以目录作为目标定位, 尝试查找路径下的 index 文件
      3. 三方模块: 先从当前路径下的 node_modules 中查找, 逐级查找上层路径的 node_modules
    3. 模块加载顺序
      1. 因为 module.exports 对象的缓存特性, 依赖模块被多次加载时, 也仅会执行一次
      2. 模块循环依赖时, 加载顺序以深度优先, 如 (main->[a,b], a->c, b->c), 因为 require 大多写在开头, 因此执行顺序为 c->a->b->main
    4. 示例
      ```
        module.exports = { name: '' }      // a.js
        let { name } = require('./a.js')   // main.js

        // node 在编译时, 对 模块 进行了包装
        (function(exports, require, modules, __filename, __dirname) {
          module.exports = { name: '' }
        })
      ```

  2. ESM (ES6 Modules): import + export  https://blog.csdn.net/weixin_33816300/article/details/91466271
    1. 特性 编译时加载 (静态加载, 模块路径必须是字符串常量, 不能是表达式)
      1. 只读引入: import 引入的变量具有 const 的特性
      2. import提升: import 语句会提升到模块开头执行
      3. 输出值引用: 模块导出后, export属性值的变化依然能影响外部模块
    2. 示例
      ```
        // index.mjs
        import { counter } from './mod.mjs'
        counter = {}; // TypeError: Assignment to constant variable.
        console.log(counter)

        // mod.mjs
        export let counter = {
          count: 1
        }
        setInterval(() => {
          console.log(counter.count)
        }, 1000)
      ```
  3. import() 动态加载
    为了弥补 import 静态加载的缺点 (按需加载、条件加载、动态路径), 提供了 import()
    相比 commonJS 的同步加载, import() 是异步加载, 返回一个 promise

  - AMD (RequireJs): define(id, [depends], callback) + require([module], callback)
    ```
      // 第一个参数为依赖数组, 第二个参数为依赖加载完成后的回调函数
      define('foo', ['jquery'], function ($) {
        // 对外暴露
        return {
          name: 'xiaoming',
        };
      });
      // 引用模块
      require(['foo'], function(foo) {
        //...
      })
    ```
    - 特点:
      1. 异步加载依赖模块
      2. 等待所有依赖加载完成再执行回调
  
  - CMD (SeaJs): define / module.exports + require
    ```
      define((require, exports, module) => {
        const foo = require('./modules/foo');
        module.exports = { name: 'xiaoming' };
      })
    ```
    - 特点:
      1. 异步加载依赖模块
      2. 按需加载, 用到时再手动 require, 而不像 AMD 一开始就加载所有依赖


