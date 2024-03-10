## 常见错误类型 https://zhuanlan.zhihu.com/p/76631124
  1. Error 所有错误的基类
  2. ReferenceError 引用错误, 如
    console.log(a) // 变量未定义
    fn() // 函数未定义
  3. InternalError JS引擎异常时由浏览器抛出的错误类型, 如栈溢出
    function fn() { fn() }; fn()
  4. SyntaxError 语法错误, 如 
    const a = ;
  5. TypeError 类型错误, 调用了对象下未定义的函数也会抛出该异常, 注意与函数变量的区别
    const o = {}; console.log(o.a.b) // 长路径取属性, 把 Undefined 当作了 Object
    window.f() // 调用了对象下不存在的方法
  6. RangeError 数组越界
    const a = 1; a.toFixed(1000)
  7. URIError 向全局 URI 处理函数传递一个不合法的URI, 较少遇到
    decodeURIComponent('%'); // URIError
    decodeURIComponent(''); // 这种不报错，大多数字符串都不报错，这部分暂时没有找到说明
  8. EvalError 使用eval函数抛出的异常, 极少遇到, 且eval执行时也会触发其他类型的错误, 如 
    eval('const a = ;'); // SyntaxError

  // 以下未测试
  9. ResourceError 资源加载错误
    new Image().src = '/remote/image/notdeinfed.png';
  10. HttpError Http请求错误
    fetch('/remote/notdefined', {})

## 捕获异常 的 API
  try { ... } catch (e) { console.log('catch', e) }
  window.onerror = (msg, url, r, c, eStack) => { console.log('onerror', url, r, c, eStack) }
  window.addEventListener('error', (error) => { console.log('error', error) })

  window.onunhandledrejection / window.addEventListener('unhandledrejection')

## 各类错误被捕获的情况 (需要写个html测试，控制台不生效)
                    try/catch  window.onerror  window.addEventListener('error')
  SyntaxError       x          x               x
  ReferenceError    √          √               √
  TypeError         √          √               √
  InternalError     √          √               √
  RangeError        √          √               √
  URIError          √          √               √
  throw 主动抛出      √          √               √
  eval 造成的错误     √          √               √

  setTimeout        x          √               √
  promise 异常       x          x               x

  
  资源加载异常                    x              √    https://zhuanlan.zhihu.com/p/351620036
  fetch 异常                                    x
  xhr 异常

  DOM操作 异常

  小结: 
    1. 语法错误 无法被捕获
    2. 异步任务 无法被 try/catch 捕获
    3. 微任务 无法被 try/catch  window.onerror  window.addEventListener('error') 捕获
  
## 框架中的错误捕获
  1. Vue: 
    Vue.config.errorHandler = () => {}
  2. React:
    componentDidCatch 钩子

## xhr 异常捕获
  1. xhr 的使用
    ``` 
      const xhr = new XMLHttpRequest();
      xhr.open('get', 'www.baidu.com')
      xhr.send();
    ```

## sentry: 
  1. 过滤器

  2. 捕获器: 用 高阶函数 包装 原生函数 (函数劫持)
    1. 包装哪些函数

    2. 包装后的作用


## 参考
  https://mp.weixin.qq.com/s/PQL6_FXzAM3eeQF2a9OsAg
  https://mp.weixin.qq.com/s/d-P8s51U6IfJ-VrRkGygLA