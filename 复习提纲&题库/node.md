## 开发框架
  - express: 对 node.http 回调的封装, 提供了 路由、中间件、模版引擎 等功能。
  - koa: 相比 express 仅保留中间件逻辑，不内置 路由、模版引擎 等功能。中间件采用洋葱模型，同时在异常处理上更友好。
  - egg: 对 koa 的封装，MVC 开发模式，约定大于配置，提供 Loader 加载并整合特定目录/名称的文件，提供插件机制作为扩展配置的单元，相比直接修改配置或者增加中间件，插件提供是否启动、加载顺序等精确控制。
  - nest: 通过模块和装饰器处理事务，依赖注入和控制反转的思想，与 egg 相比分别跟 springboot 和 springmvc 类似
  
  - era: 与 egg 功能类似，内置了美团的企业级插件。
    - sequelize: 数据库操作

## node 模块
  - node 中的模块分为 核心模块、文件模块
    核心模块: node内置模块(如: path), 在node源码编译时就被编成了二进制执行文件, 在node进程启动时直接加载进内存, 速度最快
    文件模块: 用户编写的模块, 引入时需要经过 文件定位、编译、加载进内存, 相比核心模块多了两个步骤, 速度相对慢

  - require(path) 加载模块时会调用 Module._load 静态方法, 加载过程为
    缓存优先: 尝试从 Module._cache 中加载该模块, 如有则直接返回其 exports 对象
    核心模块: 若缓存中没有, 尝试从 核心模块map映射 中获取 path 对应的模块, 如有则直接返回其 exports 对象
    用户模块: 若核心模块中未找到, 则为该文件创建 Module 实例对象, 并添加 module 到 Module._cache 中
      module.load(filename) 完成文件定位, 调用编译函数
      module._compile 在模块外层包裹一层立即执行函数, 形如
      ```
        (function(exports, require, module, __filename, __dirname) {
          // 用户模块内容
          var str = '你好';
          function hello(name) {
            console.log(str+','+name+'!');
          };
          module.exports = hello;
        })(Module.exports)
      ```
  
  - [核心模块列表](https://nodejs.cn/api/async_hooks.html)
    - assert 断言
    - async_hooks 异步钩子
      特点:
      应用场景:
      实现:
      参考资料:
        [深入理解Node.js的Async hooks](https://zhuanlan.zhihu.com/p/398853897)
        [使用 Node.js 的 Async Hooks 模块追踪异步资源](https://zhuanlan.zhihu.com/p/346978980)
    - process 进程
    - child_process 子进程
    - cluster 集群

## node 的组成
  1. v8 引擎 
  2. libuv 事件循环、线程池
  

## V8
  - V8 引擎的主要模块
    - 转换器 Parser: 将 js源码 转为 AST

    - 解释器 Interpreter: 将 AST 转为 字节码, 解释执行字节码, 收集编译器所需的 优化信息

    - 编译器 compiler: 将 字节码 转为优化后的汇编代码, 给CPU执行

    - 垃圾回收
      V8中, js对象以堆的形式存放
      V8存在对内存的使用限制, 原因是垃圾回收机制会暂停JS线程, 内存过大会增加回收耗时, 利用堆外内存可以突破限制(如buffer)
      V8的垃圾回收机制基于分代式回收机制, 根据对象存活时间分为 新生代(比如局部变量) 与 老生代(比如全局变量), 并使用不同的算法进行回收
        新生代采用 scavenge 算法, 将堆内存分为 from 和 to 两部分, 将存活对象由 from 移到 to, 之后两个空间角色互换, 以空间换时间, 过程中会存在向老生代晋升的情况
        老生代采用 标记清除(常用, 标记存活对象, 清理死亡对象) 和 标记整理(内存不足以支撑新晋对象的分配时 采用, 将存活对象往一侧移动, 以解决标记清除的空间碎片化问题)
        从根节点出发(window), 向下访问, 能访问的到即为存活
      如何高效使用内存
        多使用局部变量代替全局变量(因为会作为新生代很快被回收, 闭包的情况例外)
        主动释放老生代变量, 可通过 delete 或者 重新赋值(更好) a = null/undefined 解除引用关系
        谨慎对待闭包的使用(会造成内部函数的定义作用域无法释放, 因此要避免定义作用域中变量缓存的持续增长)
  
  - 编译阶段
    - 词法分析
    - 语法分析
    - 字节码生成

    - 作用域

    - 即时编译(JIT)

  - 执行阶段
    - 执行上下文
    

## 题库
  1. node 中的事件循环跟浏览器中有什么不同 (见 JS基础-事件循环)

  2. commonJS 的模块加载流程

  3. node 执行流程
    1. 初始化阶段
      初始化 V8 实例
      配置 libuv
      初始化 Node 实例
      运行 Node 实例
      回收 V8 实例
    2. 
