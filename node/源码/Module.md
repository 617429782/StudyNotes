## 前置知识
  [编译原理.md](../../计组&编译原理.md)

## 模块分类: 核心模块、文件模块
  - 核心模块: node内置模块(如: path)，在node源码编译时就被编成了二进制执行文件，在node进程启动时直接加载进内存，速度最快
    - JS编写的核心模块 (node/lib)
      在编译 C/C++ 模块前，会用 v8 提供的 js2c.py 将 JS代码 编译成 ASCII码，写入 node_javascript.cc
    - C/C++ 编写的内建模块 (node/src)
      内建模块，通常不会被用户直接调用，编译时也被编译成二进制指令，node 进程启动时加载进内存
    
  - 三方模块: 用户编写的模块，引入时需要经过 文件定位、编译、加载进内存，相比核心模块多了两个步骤，速度相对慢

## 模块加载: require(path): 缓存优先 -> 核心模块 -> 文件模块(文件定位 + 编译)
  require(path) 加载模块时会调用 Module._load 静态方法，加载过程为
    缓存优先: 尝试从 Module._cache 中加载该模块，如有则直接返回其 exports 对象
    核心模块: 若缓存中没有，尝试从 核心模块map映射 中获取 path 对应的模块，如有则直接返回其 exports 对象
    文件模块: 若核心模块中未找到，则为该文件创建 Module 实例对象，并添加 module 到 Module._cache 中
      module.load(filename) 完成文件定位，调用编译函数
      module._compile 在模块外层包裹一层立即执行函数，形如
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

## Module 构造函数
  ```
    function Module(id = '', parent) {
      this.id = id;
      this.path = path.dirname(id);
      this.exports = {};
      this.parent = parent;
      updateChildren(parent, this, false); // 创建模块时 将 模块自身 添加到 父模块的 children 列表中
      this.filename = null;
      this.loaded = false;
      this.children = [];
    }
  ```

## Module.prototype.require

## Module._load 静态方法

## Module.prototype.load

## Module.prototype._compile

## Module.runMain