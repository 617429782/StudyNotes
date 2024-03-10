## 第一部分
### 第一章 作用域：
    代码编译的三个阶段
      词法分析: 将 字符串分解 成有意义的代码块, 即 token 列表
      语法分析: 将词法分析的结果转换成 抽象语法树AST
      代码生成: 将 抽象语法树 转为 可执行代码

    引擎：负责程序的编译与执行
    编译器：负责 词法分析、语法分析、代码生成 (得到可执行代码)
    作用域：负责收集并维护变量组成的查询, 确定当前执行的代码对变量的访问权限

    编译阶段、运行阶段
      var a = 1
      上文代码分为两个阶段, var a 的声明阶段 和 a=1 的赋值阶段
      编译器在编译时遇到 var a, 则询问作用域是否已声明过, 如是则忽略 (多次声明同一变量只有一次生效)
      引擎在编译完成后运行代码,  遇到 a=1, 查询作用域是否有该变量 (变量提升), 如无则向上级作用域继续查找, 若直到顶级作用域依然没有找到, 则抛出异常

    LHS查询 和 RHS查询
      引擎在查询变量时, 查询的方式会影响最终的结果, RHS 查询不到会抛出 ReferenceError 异常, LHS 查询不到会在顶级作用域声明一个
      LHS: 变量赋值操作 (函数声明不属于 LHS, 而是 RHS)
      RHS: 取值操作

###  第二章 词法作用域
    词法作用域仅跟函数声明的位置有关, 而与其被调用的时机和方式无关
    词法作用域存在逐层向上查找的特性
      function foo() { console.log(a) } // 1, 因为词法作用域只跟声明的位置有关, 其声明在全局作用域下
      function bar() { var a = 2; foo() }
      var a = 1; bar()
    eval 和 with 可以欺骗词法作用域, 但会使得引擎无法在编译时对作用域查询进行优化, 进而降低性能, 不推荐使用

###  第三章 函数作用域 与 块作用域
    函数声明 与 函数表达式
      具名函数表达式 的函数名仅能在函数体内访问
        (function fn() { console.log(fn) })();  // f fn() {...}
        console.log(fn) // ReferenceError
      函数的声明不在第一位的都属于函数表达式, 如
        var a = function b() {};
        console.log(a, b) // b is not defined, b 仅在函数内部可以被访问
      使用 具名的函数表达式 是最佳实践
    概述：
      1、利用 作用域的遮蔽 遵循 最小暴露原则 是良好的设计理念
      2、作用域产生的方式有 函数、ES3的catch语句、ES6的let和const所处的块级作用域

###  第四章 变量提升
    编译阶段, 引擎会(先于执行)处理所有 变量和函数 的 定义声明
    函数表达式不会提升
    存在重复声明时, 函数优先提升 (变量的声明被忽略)
    
###  第五章 作用域闭包
    当函数类型当值被传递并在别处被调用时, 即形成了闭包, 典型的示例
      作为参数传递的 回调函数
      以函数作为 return 值

  附录A 动态作用域

  附录c this
    箭头函数的this取决于其词法作用域

第二部分 this和对象原型
  第一章 关于this
    this的指向取决于函数的调用方式, 于运行时绑定, 与函数声明位置无关

  第二章 this全面解析
    默认绑定: 全局对象(严格模式除外, TypeError)
    隐式绑定: 指向调用者(上下文对象)
      隐式丢失: 函数的引用容易丢失this, 常见如引用赋值、回调函数, 因为引用的是函数本身, 不包括上下文
        function fn() { console.log(this) }
        function fnn(f) { f() }
        var o = { fn: fn }
        o.fn()  // o
        fnn(o.fn) // window
    显式绑定: call、apply、bind
      手写一个bind (这并不是真实的实现, 没有考虑到new的情况)
        Function.prototype.bind = function(fn, caller) {
          return function() {
            fn.apply(caller, arguments)
          }
        }
      最接近的 bind 实现
        Function.prototype.bind = function(oThis) {
          var args = Array.prototype.slice(arguments, 1) // 去掉第一个参数, 即this对象
          var fToBind = this // 被调用的函数
        } 
    new绑定: 指向新创建的实例对象
    四种绑定方式的优先级 new > 显式 > 隐式 > 默认绑定
      new > 隐式
        function fn() { this.name = 1; }
        var o = { name: 123, fn: fn }
        var a = new o.fn()
        o.name  // 123
        a.name  // 1
      new > 显式
        function fn(name) { this.name = name }
        var o = {}
        var bar = fn.bind(o)
        bar(2);
        console.log( o.name );    // 2, 显式绑定修改this为 o
        var baz = new bar(3);
        console.log( baz.name );  // 3, new绑定修改this为 新实例 baz
        console.log( o.name );    // 2, bar(3) 因为bind的作用应当修改其this, 即o的name属性, 但因为new关键词而没有生效

第一章 类型
  七种类型, 原始类型 和 引用类型
    Null
    Undefined
    Boolean
    Number
    String
    Object
      Array
      Function
    Symbol
  
  typeof 存在的几个问题
    typeof null;  // object, 不能识别出 null 类型
    typeof [];  // object, 不能识别出 数组类型
    function fn() {}; typeof fn;  // function 而非 object
    typeof a; // undefined 而非 a is not defined, 该保护机制可以用于判定变量是否被定义过
  
第二章 值
  原始类型的值是不可变的, 典型的是字符串, 其成员函数并不改变原始值, 而是返回新的字符串
    var s = 'abc'; s[1] = 'o'; // s === 'abc'
  字符串反转
    因为字符串无法改变, 因此通过 Array.prototype.reverse.call 的方式来反转字符串并不可行, 只能 s.split('').reverse().join('')
  0.1 + 0.2 !== 0.3
    该问题出现的原因是js采用的 IEEE754规范 的精度问题
  void 让表达式不返回值, 或者说返回 undefined
    var a = 1; void a;  // undefined
  isNaN 的怪异行为, 对传入的值会先尝试转成数值再判断, ES6 中的 Number.isNaN 修复了这个问题，其只判断是否是 NaN
    isNaN(NaN)  // true
    isNaN(undefined)  // true
    isNaN('a')  // true
    isNaN({})   // true
    function f(){}; isNaN(f)  // true
    isNaN([])   // false
    isNaN('')   // false
    isNaN(1)  // false
    isNaN(null)  // false
    isNaN(false)  // false
  === 的怪异行为, ES6中的 Object.is 修复了这个问题, 但因为执行效率问题, 还是要尽可能使用 ===
    +0 === -0 // false
    NaN === NaN // false
  引用类型指向的是值本身, 因此一个引用不会修改另一个引用的指向
    var a = [1,2,3]
    var b = a
    a.push(4)
    b // [1,2,3,4]
    a = [4,5,6]
    b // [1,2,3,4]
  
第三章 原生函数
  