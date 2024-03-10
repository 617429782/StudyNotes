## h5-node 架构概述
  IronBase -> YouzanFramework -> Astroboy -> koa

  - Astroboy 扩展koa对象、配置约定(插件化)、MVC模型
    - 提供了对 koa 四大对象 application, context, request, response 的扩展能力, 修改了其原型
        对 controller 也提供了扩展能力, 但只是把一些常用方法从 this.ctx 提到了 this 下
        修改最多的主要是 context, 加了许多常用属性和方法

    - 约定了目录结构、命名规范, 以及部分编码方式(如路由)等, 因为他会调用 loader 按照固定的规则去加载，这些目录包括

      config、plugins、loaders、中间件、routers

    - 在 koa 洋葱模型之上，结合传统的 MVC 模型分层处理每条请求
      controller 主要做 参数校验、接口聚合
      service 主要负责业务处理，但这层很薄，业务大部分还是放在了后续的 java 层
      tether 代理层 负责dubbo服务发现、协议解析、长链接建立、负载均衡，并向上层提供http接口

    - 插件化: 提供了类似mixin的能力，将功能按相似的目录拆分到各个 plugin 中
      插件可以提供 service、中间件、工具库，也可以扩展 koa 内置对象
      plugin 可以使用 npm包，或者 指定path

    - 内置插件
      - 路由: 基于 koa-router
      - 模版引擎

  - YouzanFramework 继承于 Astroboy
    - 创建 @youzan/runtime 实例, 挂到 global.runtime
      - 创建 apollox 实例, 挂到 global.runtime.apolloClient
    - 

  - IronBase 继承于 YouzanFramework
    - 内置 业务相关的 service
    
## 启动流程
  - 1. 创建 Astroboy 实例 astroboy
    - astroboy.init
      1. 创建 koa 实例, 挂到 astroboy.app 下
      2. 创建 CoreLoader 实例, 载入代码
        - 确定各层根目录，以继承关系分层 loadCoreDirs -> astroboy.coreDirs
          依次为以下项目所在的磁盘路径: astroboy、youzan-framework、iron-base、guang-h5-node
        - 获取插件配置 loadPluginConfig -> astroboy.pluginConfig
          require 读取上述各项目中 /config/plugin.default.js (默认, 视环境而定 pre、qa 等), 汇总成一个总的 pluginConfig
        - 获取插件路径 loadFullDirs -> astroboy.dirs
          根据 pluginConfig 与 核心目录, 确定所有的插件目录
        - 获取加载器配置 loadLoaderQueue -> astroboy.loaderQueue
          与插件类似, require 读取上述各项目 /config/loader.default.js, 汇总成 loaderConfig, 根据权重从小到大排序，权重小的先注册，先执行
        - 获取加载器 loadLoaders -> astroboy.loaders
          依次 require 加载器代码, 主要有
          - AstroboyPkgLoader: 读取 package.json, 挂到 astroboy.app.pkg
          - AstroboyExtendLoader: 加载各项目中 extends 下的扩展
          - AstroboyConfigLoader: 合并所有的 /config.*.(ts|js), 挂到 astroboy.app.config
          - AutoViewLoader: 确认渲染的模板文件目录, 即各个 /views
          - AstroboyMiddlewareLoader: 拿到所有的中间件定义, 挂到 astroboy.app.middlewareQueue
          - AstroboyLibLoader: 拿到所有的工具库文件, 挂到 astroboy.app.libs
          - AstroboyBootLoader: 拿到所有的 boot.js, 应该是给 apollox 用的
          - AstroboyControllerLoader: 拿到所有的 controllers 文件, 挂到 astroboy.app.controllers
          - AstroboyServiceLoader: 拿到所有的 services 文件, 挂到 astroboy.app.services
          - AstroboyRouterLoader: 拿到所有的 routers 文件, 挂到 astroboy.app.routers, 按照固定的格式解析路由配置文件
          - AstroboyVersionFileLoader: 拿到所有的version.json, 挂到 astroboy.app.versionMap
          - AstroboyFnLoader: 拿到所有 /fns 下的文件，挂到 astroboy.app.fns
        - 执行加载器 runLoaders
          遍历加载器, 执行其 load() 方法
        - 注册中间件 useMiddlewares
          - astroboy.app.middlewares 所有的中间件
            - astroboy-static
            - astroboy-security-csp
            - astroboy-security-csrf
            - astroboy-security-cto
            - astroboy-security-frameOptions
            - astroboy-security-hsts
            - astroboy-security-p3p
            - astroboy-security-xss
            - astroboy-security-xssProtection
            - astroboy-router
            - astroboy-body
            - youzan-health-check
            - youzan-cat
            - astroboy-simple-logger
            - platform-init
            - session-init
            - progress-catch
            - error-handler
            - h5-traffic-identify
            - youzan-kdtId-check
            - init-ua-session
            - weapp-auth
            - set-hummer-config
            - dev-logger-after
            - dev-logger
            - extend-http-headers
            - guang-user
            - is-mpcrawler
            - local-session
            - logger
            - weapp-config

          - astroboy.app.middlewareQueue 当前启用的中间件
              youzan-health-check 0  忽略，只对 path 为 /_HB_ 的请求有用
              dev-logger 1  在终端打印错误日志
            - astroboy-static 3 
            - astroboy-body 15
            - youzan-cat 99
              extend-http-headers 99 扩展请求头参数，加 traceId
            - init-ua-session 101
            - weapp-config 102
            - dev-logger-after 102
              guang-user 103 根据cookie中的sessionId，调用后端接口拿到 用户信息，挂载到 ctx 上
              is-mpcrawler 240 扩展请求头参数，判断是否来自微信爬虫
            - youzan-kdtId-check 295
              set-hummer-config 296  为渲染模版 提供 hummer.config
            * logger 296 日志中间件
            * astroboy-router 300 路由中间件

          遍历 astroboy.app.middlewareQueue, 调用 koa 的方法 astroboy.app.use 注册中间件
          
    - astroboy.start
      - 启动 koa 服务, 监听指定端口
      - 触发 start 事件
      - 注册 error 事件

  - 2. 监听Apollo配置变化, 这部分看不出太大的作用
    - initUrlApollo、initDangerousScripts
      为 apolloClient 添加 命名空间监听对象, 监听远端服务器的配置变化

## 装饰器路由
  1. 阿童木框架约定了路由的配置方式，会通过loader加载所有 /routers/ 下的文件
  2. 每个 router 文件 export 出来的内容都应遵循如下格式
    [
      ['POST', '/max/weapp/authorize.json', 'account.AccountController', 'login']
    ]
    即 一个二维数组，数组中每一项代表一条路由
    每条路由包含 请求类型、URL、controller的相对路径、controller中真正执行的回调函数名
  3. 遵循这个机制，在 /routers/ 下添加了一个特殊配置文件，导出的二维数组不是静态的，而是通过 扫描函数 来生成
  4. 扫描函数 遍历所有的 Controller 文件，拿到 Controller 原型下挂载的 路由列表
  5. 路由列表 来自各个 Controller 中编写的装饰器函数，在 ts 编译阶段这些装饰器函数将被执行
  6. 装饰器函数执行时，能拿到我们传入的 请求类型、url 参数，以及 被装饰的函数函数名
  7. 结合 4和6，4中遍历时可以拿到 controller 的相对路径，加上6中拿到的参数，凑齐 2中所需的参数
  注: 阿童木loader实际的加载循序，controller 在 router 之前，因此上述序号不代表执行顺序