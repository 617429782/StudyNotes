## 日志、埋点、监控、链路追踪 的区别 https://zhuanlan.zhihu.com/p/343192736
  - 日志
    回收站: 

  - 埋点


  - 链路追踪


  - 监控: 
    定期体检, 关注特定指标, 形成告警
    低频、定期、定量

## node端的日志上报 https://naxg5qz90v.feishu.cn/wiki/wikcnBW2AYaJBBSp7nd53BqukWe#
  - 当前node端错误处理架构简述
    ```
      async function initUaSession() {
        // 有赞内置的错误处理中间件, h5-node 在配置层面去掉了, 可以不用管, 其他项目视情况
      }

      async function logger() {
        try {
          await controller()
        } catch (e) {
          console.log('中间件捕获到异常', e)
          throw e;  // 继续外抛
        } finally {
          // 上报到天网
        }
      }

      async function controller() {
        const result = await service(); // 如果没有throw, 返回的是 undefined, 有的话将报错被中间件捕获
        // 也就是说, 如果 dubbo返回的是拒绝态, controller 应该是拿不到拒绝原因的, 除非在 controller 中再次catch
        this.ctx.success(result);
        console.log('controller', result);
      }

      async function service() {
        return await invoke();
      }

      async function invoke() {
        try {
          return await dubbo();
        } catch(e) {
          // 上报到天网
          throw e; // 如果不往外抛出异常, 则 controller、logger 都认为没有异常, 那么就不会触发 logger 中的catch捕获
        }
      }

      function dubbo() {
        return new Promise((ro, rj) => {
          setTimeout(() => {
            // ro({ msg: 'ok' });
            rj({ msg: '网络异常' });
          }, 500)
        })
      }

      logger()
    ```
    因为 koa 洋葱模型的执行特点, logger 中间件应该在其他中间件之前执行, 即优先级应该设为最高

  - 当前架构存在的问题
    - 当前invoke 调用 service 接口 catch 到异常时, 会继续 throw, 不 catch 的话会在 中间件层再次上报, 即一次异常上报了两次
      1. invoke 中 catch 掉 dubbo 的异常后, 需要 throw, 因为 报错原因需要携带到客户端
      2. 基于第一点, controller 中调用 service 接口时, 不要加 .catch(noop), 这会吞掉 throw 出来的 错误原因
      3. 基于第二点, controller 中应该用 try/catch 包裹, 并在 catch 中向客户端返回错误信息
      4. 基于第三点, 运行时的语法错误不应该被返回到客户端, 因此只应该对 调用service接口 的部分用 try/catch 包裹
      5. 基于第四点, 为保证运行时的错误不中断程序, 应该再在 controller 中套一层 try/catch, 并给客户端返回一个通用性提示

      总结: 最终 controller 中的写法应该是
      ```
        async function controller() {
          try {
            let result = [];
            try {
              result = await Promise.all([
                service1(),
                service2()
              ])
            } catch (e) { // 这一层是为了捕获 invoke 抛出的错误原因, 如 '津贴已领完', 应直接返回给 客户端
              ctx.json(e.code, e.message);
            }
            ctx.success(result);
          } catch (e) { // 这一层是为了捕获运行时的语法错误, 应该给客户端返回一个通用提示
            ctx.json(9999, '网络异常');
            throw e; // 由中间件捕获上传到天网, 或者手动调用 this.logError, 这样就不用抛出异常
          }
        }
      ```
      6. 或者, 可以只用一层 try/catch 捕获异常, 判断 e.code 来决定后续处理
        ```
          async function controller() {
            try {
              const result = await Promise.all([
                service1(),
                service2()
              ])
              ctx.success(result);
            } catch (e) {
              if (e.code) { // 有 code 认为是服务端返回的 业务异常 或 网络问题, 这里可以根据具体值做细分处理
                ctx.json(e.code, e.message);
              } else {  // 这一层是为了捕获运行时的语法错误, 应该给客户端返回一个通用提示
                ctx.json(9999, '网络异常');
                throw e;
              }
            }
          }
        ```
      7. 这里有一个细节, 业务异常返回给客户端的code不能给0, 这个取决于客户端对请求的处理方式, 下文展开讲
    
    - 存在部分被归类到 warning 等级, 如 [JS代码异常]、部分 dubbo 报错

    - 存在部分未格式化就上报的日志, 如直接调用 ctx.logger.info, 这部分可能有出于业务性的考虑, 需要更全面的了解

    - 部分超时的接口, 错误信息会返回 {"code":400,"msg":"网络错误, 请稍后再试","data":null}

    - 客户端对错误请求的处理
      - 在node端发生异常时返回给客户端的错误码不一致, 有些返回的是0, 有些返回的是 9999, 还有其他的值 500

      - h5
        - common/request -基于zan-ajax的封装, 返回的code为0时才进入 then, 否则抛出异常进入 catch
        - 部分采用了自己封装的 request, 也是对zan-ajax的封装, 如 
          activity-stage/api (搜 request as zanRequest) @王建
          requestMixin (搜 requestMixin)  @赵健

      - 小程序

      - react


  // 以下部分偶发, 先观察后续有无出现
  - todo: 存在部分接口堆栈定位到 /@youzan/youzan-framework/plugins/youzan-dubbo/app/lib/DubboClient.js
    本地调试时, 这个文件中的接口并没有被调用, 需要排查此类日志产生的原因


  
      