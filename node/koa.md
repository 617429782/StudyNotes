## koa 概述
  - 相比于 express，koa 崇尚的 async / await 更贴合同步的程序运行思维
  - koa 提供了更好的错误处理
  - koa 只保留了核心的中间件逻辑，不内置路由、模版渲染等功能，而是通过中间件来实现，使得核心代码更加精简，同时开发者拥有更多的选择权

## [中间件](https://juejin.cn/post/6854573208348295182)
  - use/callback 发布订阅模式
    app.use 将中间件回调存入 app.middleware
    app.callback 调用中间件队列的入口函数
    
  - 洋葱模型: 递归/函数栈
    入口函数 将ctx和下一个中间件作为传给 第一个中间件回调，形成一个递归的调用
    类似函数调用栈，先从外向内执行所有中间件 next 之前的部分
    然后从内向外执行所有中间件 next 之后的部分
    每个中间件回调会被执行两次，因为回调参数包含ctx，因此任意顺序的中间件都有机会修改其他中间件的处理结果

  - 核心代码
    ```
      const middleware = [];

      function use(cb) {
        middleware.push(cb)
      }

      function callback() {
        return componse();
      }
      
      function componse(ctx) {
        function dispatch(i) {
          const fn = middleware[i];
          if (i === middleware.length) return;
          return fn(
            ctx,
            dispatch.bind(null, i+1)
          )
        }
        return dispatch(0)
      }

      use((ctx, next) => {
        console.log(1)
        next();
        console.log(4)
      })

      use((ctx, next) => {
        console.log(2)
        next();
        console.log(3)
      })

      callback();
    ```
    源码中，componse的两种返回值都用了 Promise.resolve() 包裹，这是为了支持 next().then() 的写法

## 上下文 context
  - http.createServer 创建服务后的回调接收 req 和 res 两个参数
  - koa 将这两个参数包装成了ctx
    ```
      // app.listen(3000)
      listen(...args) {
        const server = http.createServer(this.callback());
        return server.listen(...args);
      }
      callback() {
        return (req, res) => {
          // 基于 node 原生的 req 和 res 创建 ctx
          const ctx = this.createContext(req, res);
          return this.handleRequest(ctx, compose(this.middleware));
        };
      }
    ```
  - 为了方便操作 context 上同时也挂载了 request、response 对应方法的别名, 利用 apply 指定this

## 错误处理

## 内容协商

## 缓存刷新

## 代理支持

## 重定向

## koa-router
  - Layer: 单个路由的管理，包含 url、method、callbak，提供 路由匹配、query 解析等方法
  - Router: 所有路由的管理
