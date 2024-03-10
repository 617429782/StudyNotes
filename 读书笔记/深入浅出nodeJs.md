## 模块
  node中模块的实现思路是 找到文件后，创建 Module 实例，并为其添加 脚本执行结果 作为缓存
  Module 的简易实现
  ```
    function Module (id, parent) {
      this.exports = {};
      this.load = function(path) {

      }
    }

  ```

  ```
  var str = '你好';
  function hello(name) {
    console.log(str+','+name+'!');
  };
  module.exports = hello;
  ```
  上述代码将被立即执行函数包装，形成独立的作用域, 这也是变量 exports, require, module 未定义却可以被访问的原因
  ```
  (function(exports, require, module, __filename, __dirname) {
    var str = '你好';
    function hello(name) {
      console.log(str+','+name+'!');
    };
    module.exports = hello;
  })(Module.exports)
  ```

## 异步I/O
  异步的优势体现在:
    并发的方式使得响应总时长变短，用户体验更佳
    即绕过了多线程中死锁、状态同步等问题，又没有同步单线程中，I/O 阻塞计算任务导致的性能问题