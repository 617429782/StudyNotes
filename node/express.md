## 概述
  node 创建 http 服务的基本代码
  ```
    const http = require('http');
    const server = http.createServer(function(req, res){
      res.end('hello word!')
    });
    server.listen(8000);
  ```

  express 创建了 http.createServer 的回调函数, 在 res.end 响应客户端请求之前 顺序执行中间件注册的回调
  ```
    const express = require('express')
    const app = express()
    const port = 3000

    app.listen(port, () => {
      console.log(`Example app listening at http://localhost:${port}`)
    })
  ```

  其中 express 和 app.listen 的简易实现
  ```
    var express = createApplication = () => {
      var app = function(req, res, next) {
        app.handle(req, res, next);
      };
      app.handle = (req, res, next) => {
        // ...下文实现
      };
      app.listen = () => {
        var server = http.createServer(this); // 注意这个this, 等同于 http.createServer((req, res, next) => { app.handle(req, res, next); })
        return server.listen.apply(server, arguments);
      };
      return app;
    }
  ```

  小结: express 是对 node.http 回调的封装, 提供了 路由、中间件、模版引擎 等功能

## 中间件
  中间件是一个函数, 形参 res、req、next
  通过 app.use 注册的中间件将存在函数队列中, 并在 app.handle 顺序执行, 用代码简述一下这个过程
  ```
    var app = { queue: [] }
    app.use = function (middleware) {
      this.queue.push(middleware)
    }

    app.hanlde = function (req, res) {
      const queue = this.queue; // 使用闭包缓存队列, 避免next执行时this丢失
      // 执行完函数队列后调用 res.end
      const done = (req, res) => {
        console.log('done')
        res && res.end()
      }
      function next() {
        if (!queue.length) return done();
        const middleware = queue.shift();
        middleware(req, res, next)
      }
      next()
    }

    app.use((req, res, next) => {
      console.log('before next1')
      next();
      console.log('after next1')
    })

    app.use((req, res, next) => {
      console.log('before next2')
      next();
      console.log('after next2')
    })
  ```
  app.handle 在执行时, 可以理解为 嵌套执行 middleware, 将其不断加入执行栈, 直到 done, 之后再逐个完成执行出栈
  因此上述注册的中间件, 输出顺序为
    before next1
    before next2
    done
    after next2
    after next1
  这就是所谓的 洋葱模型, 在 next 之后逆向执行 after, 使得 之前的中间件 有办法拿到 后续中间件的处理结果

## 路由
  ```
    const router = express.Router()
    // 静态路由
    router.get('/api/getName', async (req, res, next) => {
      res.status(200).send('xiaoming =')
    })
    // 动态路由
    app.get('/product/:id/:name', (req,res) => {
      res.end(JSON.stringify(req.params));
    });

  ```

## 模版引擎