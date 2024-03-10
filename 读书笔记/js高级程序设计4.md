- 一、什么是javascript


- 二、HTML中的js

- 三、语言基础

- 四、变量、作用域、内存

- 五、基本引用类型

- 六、集合引用类型

- 七、迭代器与生成器
  1. 理解迭代
    - for循环 和 forEach 各自有一些缺陷, 如 
      for循环需要维护下标变量, 并且需要通过 数组本身 来取得迭代值
      forEach 则无法中止, 并且只能被数组调用
    - for of 一定程度上解决了上述问题, 其底层实现基于 迭代器模式

  2. 迭代器模式
    - 判断一个变量是否可迭代可以观察其是否有 [Symbol.iterator] 属性, 作为默认迭代器, 该属性为迭代器工厂函数, 返回一个迭代器
    - 对象不可迭代, 直接原因是没有[Symbol.iterator], 更深层的原因可能是对象的默认迭代行为很难定义, 如是否包含原型属性, Symbol属性、是否可枚举等, 所以迭代的逻辑交给程序员按需实现
    - 迭代器是一个对象, 持有 next 方法, 调用后返回 IteratorResult { value: [key, value], done: Boolean }

  3. 生成器
    - 生成器函数的语法是函数名前加 *, 匿名函数可以作为生成器函数, 箭头函数不能作为生成器函数
    - 执行生成器函数会得到 生成器对象, 生成器对象一开始处于暂停执行（suspended）的状态, 其实现了 Iterator 接口
    - yield 关键字可以暂停执行生成器, 保留当时的作用域状态, 暂停后只能调用 生成器对象的next方法 来恢复
    - yield 关键字可以作为变量, 用于 next() 的输入输出
      ```
        function *fn(initParam){ 
          yield initParam + 'a';
          var value = yield 'b'
          yield ('receive: ' + value);
          return 'c';
        }
        var g = fn('init: ');
        g.next(1) // { value: "init: a", done: false}  // 第一次设置值不起效, 会取 fn() 传入的值
        g.next(1) // { value: 'b', done: false }  // 第二次设置起效, 在之后的作用域中存在
        g.next(1) // { value: "receive: 1", done: false }
        g.next(1) // { value: 'c', done: true }
      ```
    - yield* 可以将一个可迭代对象转为一系列单独产出的值，其作用是将迭代控制权交给另一个迭代器
      ```
        function* fn() { 
          yield* [1, 2, 3]; 
        }
        var g = fn();
        g.next() // { value: 1, done: false } 
        g.next() // { value: 2, done: false } 
        g.next() // { value: 3, done: true } 
      ```
    - 多次执行生成器函数可以得到多个 生成器对象, 相互间作用域不影响

  4. 小结

- 八、对象、类与面向对象编程
  1. 理解对象
  2. 创建对象
  3. 继承
  4. 类
  5. 小结

- 九、代理与反射

- 十一、Promise
  1. 异步编程
    - 同步任务对应内存中顺序执行的处理器指令，执行后也能立即得到存储在本地的信息
    - Promise/A+
      三种状态 pending fulfilled rejucted，只能转换一次且只能由 pending 转成其他状态
      Promise.resolve 可以得到 fulfilled 态的实例，但若入参就是 promise，则不改变其原来的状态
      ```
        Promise.resolve(new Promise(() => {})) // Promise <pending>
      ```
      Promise.reject 可以得到 rejected 的实例并抛出一个异步的错误，其不能被try/catch 捕获，只能通过 Promise 实例的catch捕获 (或者监听特定事件)
      
  
  2. Promise

  3. 异步函数

  4. 小结

- 二十一、错误处理与调试
  1. 浏览器错误报告
  2. 错误处理
    - 八种错误类型
      Error 所有错误的基类
      ReferenceError 找不到对象, 长路径取属性、变量未定义等
      InternalError JS引擎异常时由浏览器抛出的错误类型, 如栈溢出
      SyntaxError 语法错误
      TypeError 类型错误, 调用了对象下未定义的函数也会抛出该异常, 注意与函数变量的区别
        window.f() // TypeError
        f() // ReferenceError
      RangeError 数组越界
      URIError encodeURI、decodeURI, 较少遇到
      EvalError 使用eval函数抛出的异常, 极少遇到, 且eval执行时也会触发其他类型的错误

    - try / catch / finally
      try/catch 中, catch到的错误对象 error 的 message 属性是被所有主流浏览器兼容的
      finally 语句不会被 try/catch 的代码阻止, 包括 return
      被 try/catch 捕获后, 浏览器将不再抛出异常, 可以用 throw 手动抛出触发
      没有被 catch 的错误都将触发 error事件, 可以被 window.addEventListener('error') 与 window.onerror 捕获

    - window.addEventListener('error') 与 window.onerror
      - 监听器 和 onerror 都能捕获 运行时错误(注意区分 编译时错误), 但回调接受的参数不同
        window.onerror 实现较早, 为了向前兼容返回的是 错误信息、报错url、行号、列号、error对象
        监听器 返回的是一个 event 对象实例
        ```
          window.onerror = (...args) => { console.log(args) }
          window.addEventListener('error', (...args) => { console.log(args) })
          var a = {}
          setTimeout(() => { a.fn() }, 0)
        ```
      - 监听器 可以捕获 资源加载错误, 而 onerror 不能 // todo: 未能有效验证
      - window.onerror 仅在返回 true 时错误才会继续上报

    - Promise 中发生的错误
      - 在 promise.then 中发生的错误不会被 error 捕获
      - 相对的, 可以使用 window.addEventListener("unhandledrejection") 和 window.onunhandledrejection 捕获
        ```
          window.addEventListener("unhandledrejection", (e) => {
            console.log('cyk1', e);
          });
          window.onunhandledrejection = (...args) => { console.log('cyk2', args) }

          // 手动抛出的异常也会得到一个 rejected 的实例, 但经测试只有这个方式能被捕获
          new Promise((resolve) => {
            resolve();
          }).then(() => {
            // throw new Error('123') // 经测试无法被捕获
            throw 'promise error'
          });

          // 以下两种方式经测试 都没有被捕获
          Promise.reject('promise error');
          new Promise((resolve, reject) => {
            reject('promise error');
          });
        ```
      - 迷惑点: // todo
        仅有在 then 中手动抛出的非 Error 实例被捕获到了, 但在常规的 error 事件中，new Error 也会被捕获到
        手动调用 reject 没有被捕获到

  3. 调试处理
  4. 旧版IE常见错误
  5. 小结
    - js中的错误将阻断程序继续执行，使用 try/catch 捕获潜在的异常可以提高程序的健壮性
    - 使用 window.addEventListener('error') 或 window.onerror 来捕获未加 try/catch 而遗漏的异常
    - promise 中抛出的异常不会触发 error 事件，需要单独处理
    - 错误上报常用的方式为 ajax 和 image 对象，后者的优势是无需考虑跨域

- 二十四、网络请求与远程资源
  1. XHR 对象
    - xhr 对象的使用
      ```
        const xhr = new XMLHttpRequest();   // 创建
        // 请求方法、目标url(必须同源)、是否异步
        xhr.open('get', 'xxx.com', false);  // 准备发送请求，需等到send方法调用
        xhr.send(null)                      // 发送请求，若没有发送内容必须传 null

        // xhr 会默认带上部分请求头，若需添加额外属性，使用 setRequestHeader
        xhr.setRequestHeader('token', 'balabalabla')

        // xhr 对象发生状态变化时都将触发 readystatechange 事件，异步请求可以监听该事件

        // 收到服务端响应后, 下列属性会自动被添加
        xhr.responseText
        xhr.status

        // 响应头头可以通过 getResponseHeader / getAllResponseHeaders 获取

        // 若想在收到响应前取消可以调用 abort 方法
        xhr.abort()
      ```
    - xhr 模拟表单时需要序列化内容，并设置特定请求头，为了简化操作，可以用 FormData实例 作为 send 的数据
    - 当开发者比服务端更明确返回内容的类型时，可以在 send 前调用 xhr.overrideMimeType 覆盖 response 中的 MIME 类型

  2. 进度事件
    - 监听 readystatechange 的方式并不友好，xhr 在特定阶段会触发特定事件，如
      ```
      loadstart：在接收到响应的第一个字节时触发。
      progress：在接收响应期间反复触发。
      error：在请求出错时触发。
      abort：在调用 abort()终止连接时触发。
      load：在成功接收完响应时触发。
      loadend：在通信完成时，且在 error、abort 或 load 之后触发。
      ```

  3. 跨源资源共享

  4. 替代性跨源技术

  5. Fetch API

- 二十七、工作者线程
  1. 工作者线程 简介
    - 浏览器的多个tab是相互独立的执行线程，在执行线程外，可以创建 工作者线程
    - 工作者线程包括 专用、共享、服务工作者线程
      专用工作者线程: web-work, 只能在创建该线程的页面中使用
      共享工作者线程: 可以被多个页面的脚本使用，但需遵循同源规则
      服务工作者线程: 前两者主要用于通信，服务线程负责拦截、重定向、修改页面发出的请求
    - 工作者线程中的全局对象并非 window，而是其子集 self

  2. 专用工作者线程 worker
    - web-worker 可以与主线程通信、发送网络请求、文件输入输出、处理大量数据、密集计算等
    - web-worker 的使用
      ```
        // 创建
        const worker = new Worker(filePath) // 可以是本地文件，也可以是网络文件，但必须同源
        // 通信 - 消息机制
        worker.onmessage = () => {} // 接受工作者线程通过 self.postMessage() 传过来的消息
        worker.postMessage()  // 向工作者线程发送数据，线程内需要通过 self.addEventListener('message') 监听
        worker.terminate() // 立即终止工作者线程，并不等其完成数据清理
      ```
    - worker对象不会被垃圾回收，除非线程终止，因此需对其做好管理
    - importScripts 可在工作者线程内加载其他脚本，类似 <script>, 没有同源限制，相互间共享作用域
    - try/catch 无法捕捉到worker内部抛出的异常，可以通过 worker.onerror 感知错误的抛出
    - 基于 MessageChannel 的通信，两个线程需分别持有两个通信端点，耦合性高
      ```
        const m = new MessageChannel() // 每个实例都有 port1 和 port2 两个通信端点对象，可用于收发数据
        m.port1.onmessage = () => {}
        m.port2.postMessage()
      ```
    - 基于 BroadcastChannel 的通信，同源的脚本可以各自创建 BroadcastChannel 实例，无耦合性
      注意点: 同名、同源
      ```
        // tab1
        var b = new BroadcastChannel('test')
        b.onmessage = (e) => {console.log(e)}

        // tab2
        var b = new BroadcastChannel('test')
        b.postMessage('1')

      ```
    - 跨线程的数据传输
      结构化克隆算法: 浏览器私有算法，用 postMessage 传递时在目标上下文生成传输数据的副本 (相当于深拷贝)
        该算法不克隆原型链，不支持Symbol、函数对象、Error对象、DOM节点等
        ```
          const worker = new Worker('./worker.js'); 
          // 创建 32 位缓冲区
          const ab = new ArrayBuffer(32); 
          console.log(`page's buffer size: ${ab.byteLength}`); // 32
          worker.postMessage(ab); 
          console.log(`page's buffer size: ${ab.byteLength}`); // 32

          // worker.js
          self.onmessage = ({ data }) => { 
            console.log(`worker's buffer size: ${data.byteLength}`); // 32
          };
        ```
      可转移对象: 特定类型的对象在通信时，将转移所有权 (相当于剪切+粘贴)
        这些类型包括 ArrayBuffer、MessagePort、ImageBitmap、OffscreenCanvas
        postMessage 第二个参数用于指定转移对象
        ```
          // 主线程
          const worker = new Worker('./worker.js'); 
          const ab = new ArrayBuffer(32); // 创建 32 位缓冲区
          console.log(`page's buffer size: ${ab.byteLength}`); // 32
          worker.postMessage(ab, [ab]); 
          console.log(`page's buffer size: ${ab.byteLength}`); // 0

          // worker.js
          self.onmessage = ({ data }) => { 
            console.log(`worker's buffer size: ${data.byteLength}`); // 32
          };
        ```
        postMessage 第二个参数还可以指定对象的属性为 转移对象，此时对象会被复制，其属性被转移
        ```
          const worker = new Worker('./worker.js'); 
          // 创建 32 位缓冲区
          const ab = new ArrayBuffer(32); 
          console.log(`page's buffer size: ${ab.byteLength}`); // 32
          worker.postMessage({foo: {bar: ab}}, [ab]); 
          console.log(`page's buffer size: ${ab.byteLength}`); // 0

          // worker.js
          self.onmessage = ({ data }) => { 
            console.log(`worker's buffer size: ${data.foo.bar.byteLength}`); // 32
          };
        ```
      共享对象 SharedArrayBuffer
        SharedArrayBuffer 实例作为数据传输时，两个线程各自持有内存的引用，共同维护

    - 线程池
      启用web-worker需要较大代价，可以用线程池来管理正在执行计算任务的活动线程
  
  3. 共享工作者线程 sharedWorker
    - sharedWorker 在 相同的url参数 下是 单例 的，这一点与 worker 每次都创建新实例不同
      ```
        // 以下三种情况被视为相同的url
        new SharedWorker('./sharedWorker.js'); 
        new SharedWorker('sharedWorker.js'); 
        new SharedWorker('https://www.example.com/sharedWorker.js');
      ```
      后续创建的实例，将 connect 到已有的线程 (创建时内部调用)
      可以通过第二个参数指定线程名称来绕过这个限制
      ```
        new SharedWorker('./sharedWorker.js', { name: 1 }); 
        new SharedWorker('./sharedWorker.js', { name: 2 }); 
      ```
    - 页面关闭时不会自动销毁连接时产生的 MessageChannel 实例，可以在 beforeunload 时发送消息手动卸载

  4. 服务工作者线程 ServiceWorkerContainer
    - 服务工作者线程类似代理服务器，可以拦截请求/缓存响应，使得在无网络时也可以保证一定的用户体验
    - serviceWorker 没有构造函数，用 navigator.serviceWorker.register 来建立连接
    - 

  5. 小结


async 会开始下载脚本, 但不阻塞UI线程, 等下载完成后再执行, 异步脚本会在页面的 load 事件前执行, 但可能会在 DOMContentLoaded 之前或之后

defer 会让脚本在页面完成加载后执行

typeof 的问题
  null 被判定为 object
  function 被判定为 function
  array 被判定为 []

有三种函数可以将 字符串 转为 数字
  Number(), 与 + 一元操作符遵循同样的规则
    不传, 空字符串, null、false 返回 0
    undefined 返回 NaN (跟不传返回的值不一样, 需注意)
    对于 字符串 尝试返回一个 十进制 的数值, 否则返回 NaN
    对于 对象, 优先调用 valueOf() 获取原始值 (new String(1) 类型的原始值是 '1', 因为其 valueOf 被默认改写了)
      var s = new String(1);
      s.valueOf = () => { console.log('valueOf'); return '1' };
      s.toString = () => { console.log('toString'); return '1' };
      Number(s) // valueof
    若 valueOf() 为 引用类型 再尝试用 toString() 按字符串的规则
      var s = {};
      s.valueOf = () => { console.log('valueOf'); return {} };
      s.toString = () => { console.log('toString'); return '1' };
      Number(s) // valueof toString
  parseInt(string, [option])
    转换规则为 取第一个 有效整数字符(数值、加减号) 到 第一个 无效字符, 否则返回 NaN, 第二个参数可指定进制
    由此得出: 不传、空字符串、null、undefined、Boolean 都会返回 NaN
    以0x开头时会被转成十六进制
  parseFloat(string)
    转换规则与 parseInt 类似, 取第一个 有效浮点字符(数值、加减号、小数点) 到 第一个 无效字符, 否则返回 NaN

toString 方法只在调用者为 数值 时, 才接受第二个参数

