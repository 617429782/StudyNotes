## 概述
  webpack是程序的静态模块打包器, 根据程序中的依赖关系, 递归地生成一个依赖关系图, 然后将这些模块打包合并成一个或多个bundle (chunk)

## 名词释义
  module: 即模块，可以理解为一个文件一个module
  dependency: 依赖对象，记录 module 的依赖，并在当前 module 完成解析后转为 module 递归解析
  chunk: 从 entry 出发，聚合所有 module 的 代码块, 另外 动态引入 的模块自成一个 chunk
  bundle: 

  compiler: 编译管理器，webpack 整个生命周期只创建一次
  compilation: 编译上下文，每次编译都会创建一个新对象，如 watch 配置下的每次文件改动

  moduleGraph: 记录本次编译所有的 module 和 dependency 及其关联关系
  moduleGraphConnection: 记录模块间引用关系，originModule属性 为父模块，module属性 为子模块
  chunkGraph: 

## webpack 构建流程 init、make、seal、emit
  - 初始化 init: (初始化参数、创建构建管理器和上下文、加载插件)
    初始化参数: 获取配置文件 和 shell 中得到的参数 (cli.js)，校验并格式化 (new webpack())
    初始化编译 new Compiler(): 创建 编译器 compiler 实例, 定义编译相关的 hooks
    加载用户插件: 加载用户定义的插件(option.plugins), 执行其 apply
    加载内置插件: 调用 new WebpackOptionsApply().process，该方法为 众多内置插件的执行入口，主要作用是 注册钩子
      触发 ‘entryOption’ 钩子，遍历 option.entry，为每个入口创建 EntryPlugin 实例，注册 'compilation' 和 'make' 钩子

  - 编译
    执行 compiler.run 开始编译 (cli.js)
      执行 compiler.compile
        触发 'compilation' 钩子
        触发 'make' 钩子
          调用 compilation.addEntry 在 compilation.entrys 中加一个入口 (dependence 对象)
          调用 compilation.addModuleTree
            调用 compilation.handleModuleCreation
              调用 compilation.factorizeModule
                调用 compilation.addModule 
                  遍历 dependencies，调用 compilation.moduleGraph.setResolvedModule
                  调用 compilation.moduleGraph.setIssuerIfUnset
                  调用 compilation.buildModule(module, callback)
                    调用 compilation.buildQueue.add(module, callback)  // todo: 异步队列
                      调用 compilation.buildQueue._ensureProcessing
                      调用 compilation.buildQueue._startProcessing
                      调用 compilation.buildQueue._processor
                      调用 compilation.buildQueue._handleResult
                    ...
                    // todo
                  调用 compilation._buildModule(module, callback)
                    调用 module.doBuild 
                      资源转换: 调用 runLoaders
                        调用 loadLoader，加载 loader 代码
                        调用 runSyncOrAsync，执行 loader 代码
                      依赖分析: 调用 module.parser.parse
                        调用 AcornParser.parse 拿到 ast
                        遍历 ast.body 节点，找到 Import 声明, 即当前模块的依赖
                        // todo 
                        ast 遍历完毕，调用 handleParseResult
                      递归构建: 对依赖的模块进行 build 的流程
  - 生成
      调用 compilation.finish 方法
        触发 'finishModules' 钩子，此时已经有了所有的 modules
        
      调用 compilation.seal 方法
        基于 moduleGraph 创建 ChunkGraph 对象
        遍历 compilatio.modules, 记录 module 与 chunk 的对应关系 (在 weakMap 中存入 {module: chunkGraph} )
  
        // todo 触发各种钩子 optimizeDependencies
        触发 optimizeDependencies 钩子
        触发 afterOptimizeDependencies 钩子
        触发 beforeChunks 钩子
        调用 compilation.addChunk 

        调用 buildChunkGraph
        
        // todo 触发各种钩子 optimizeModules
        // todo 触发各种钩子 optimizeChunks
        
        触发 优化钩子
        根据 entry 和 依赖关系图 将 module 分配给 chunk 对象
        遍历 chunks, 调用 emitAssets 将 assets 写入 compilatio.assets

      调用 onCompiled 回调
        调用 compiler.emitAssets
          遍历 compilation.assets, 写入文件系统

## 输出的 bundle.js 分析
  ```
    (() => {
      // 模块列表，包括业务模块，引入的三方库等
      var __webpack_modules__ = {
        // 不同模块 执行函数的参数 可能不一样
        "./src/lib/common.js": (module) => {
          eval("module.exports = { log: (msg) => { console.log('cyk ', msg) }}");
        },
      }
      var __webpack_module_cache__ = {};

      // require 函数，作用是取得 module.exports 对象
      function __webpack_require__(moduleId) {
        // 优先从缓存中加载模块，减少 eval
        if (__webpack_module_cache__[moduleId]) return __webpack_module_cache__[moduleId].exports;
        // 创建新模块并加入到缓存
        var module = (__webpack_module_cache__[moduleId] = { exports: {} });
        // 执行 eval 函数，为 module.exports 赋值
        __webpack_modules__[moduleId](module, module.exports, __webpack_require__);
        return module.exports;
      }

      __webpack_require__("./src/lib/common.js");
    })()
  ```
  1. 打包的结果是一个立即执行函数
  2. 函数内容分为 模块映射表、缓存映射表、require函数定义
  3. require函数的作用是执行 模块列表中 各模块对应的函数(作用是修改module.exports) 并将结果 module.exports 加入到缓存列表
  4. 不同的模块加载方式会让打包结果有差异，AMD模块会定义更多的 require 函数辅助执行

## 热替换原理 https://zhuanlan.zhihu.com/p/30669007
  - 概念说明
    webpack-dev-server/server: 负责创建 express 服务器提供静态资源服务，以及 socket服务端 的创建
    webpack-dev-server/client: 
      负责 socket客户端 的创建，定义消息处理事件
      收到 'ok' 时调用 reloadApp，触发 'webpackHotUpdate' 事件
    devServer: 监听 webpackHotUpdate 事件，进行 hotCheck
      使用 fetch 请求 .../index.[hash].hot-update.json 获取更新的 diff
        调用 __webpack_require__.hmrC.jsonp 请求 .../index.[hash].hot-update.js
          创建 <script> 标签，将 src 设为 .../index.[hash].hot-update.js

  - 流程: 创建服务 + 文件监听 + 推送改动 + 页面刷新
    - 创建服务，建立连接
      调用 webpack 的初始化流程，得到 compiler 对象
      let server = new Server(): 创建 两个服务器，注入客户端代码
        - addEntries 在打包结果中注入 热更新客户端运行时，主要包含
          hotEntry: dev-server
          clientEntry: webpack-dev-server/client

        - express 提供静态资源服务 (记为 server.listeningApp)
          setupApp(): 创建 express 服务器
          setupDevMiddleware(): 设置 express 中间件
          routes(): 设置 /webpack-dev-server/ 相关路由
          setupFeatures(): 主要是 express 服务器的相关配置

        - socketServer 提供 update chunk hash 的推送服务
          创建 SockJsServer 实例
      
    - watch
      - webpack 持续监听 文件编辑时间，监听到文件变化后触发 模块构建流程
        常规webpack打包后的文件是直接写入文件系统的 (compiler.outputFileSystem = require("graceful-fs"))
        热更新下，webpack-dev-middleware 将 compiler.outputFileSystem 改成了 MemoryFileSystem，将 chunk 写在内存中
        写在内存中的好处是读写更快，这也是热更新下，dist文件夹(output.path) 没有变化的原因
        可以设置 devServer.writeToDisk 让热更新下的 chunk 写入文件系统，以便于调试
      - 

    - send
      - 

    - reload

    执行 HotModuleReplacementPlugin 中注册的 compilation钩子回调
    执行 compiler.compile, 依次触发 make、finish、seal 相关钩子
    
## 配置优化

## Tapable https://zhuanlan.zhihu.com/p/100974318
  - 基础用法 (创建、注册、触发)
    ```
      // 创建
      const hook = new SyncHook(); 

      // 注册，注册的回调会存在 hook.taps 中, 第一个参数是回调的options，包含 name、type 等
      hook.tap('first', (msg) => { console.log('first:', msg) }) 
      hook.tap({ name: 'second' }, () => { console.log('second') }); // 第一个参数也可以是 配置对象

      // 触发
      hook.call() 

      // 创建时传入的数组，将作为 call 时的形参
      const hook2 = new SyncHook(['first']);
      hook.tap('first', (name, other) => {   // other 为 undefined，因为 创建时只声明了一个形参
        console.log(name, other);
      });
      hook.call('call', 'test');  // 即便call传了两个参数，tap注册的回调也只能收到一个
    ```
    - 创建

    - 注册

    - 触发

  - 执行顺序 stage、before
    stage 指定同一钩子多个回调的执行顺序，数字大的后执行
    before 可以指定在某个回调前执行，因此 name 属性是必填属性

  - 拦截器 intercept: 监听 事件回调的注册、调用以及事件的触发 等
    ```
      const hook = new SyncHook();
      hook.intercept({
        register(tap) { // 注册时执行
          console.log('register', tap);
          return tap;
        },
        call(...args) { // 触发事件时执行
          console.log('call', args);
        },
        loop(...args) { // 在 call 拦截器之后执行
          console.log('loop', args);
        },
        tap(tap) { // 事件回调调用前执行
          console.log('tap', tap);
        },
      });
    ```

  - hook 的类型 (回调逻辑 x 触发方式)
    - 回调逻辑
      Basic: 基础类型，仅调用回调，不关心返回值，如 SyncHook
      Bail: 保险类型，多个回调顺序执行时，若其中一个回调不返回 undefined, 则不执行后续回调，如 SyncBailHook
      Waterfail: 瀑布类型，多个回调顺序执行时，若其中一个回调不返回 undefined, 则将其作为第一个参数传给下一个回调
      Loop: 循环类型, 多个回调顺序执行时，若其中一个回调不返回 undefined, 则从第一个回调重新开始执行直到都没有返回值

    - 触发方式
      Sync: 同步触发，只能用 tap 注册，用 call 触发
      Async: 异步触发
        不能用 call 触发，只能通过 callAsync 或者 promise，可以触发 tap、tapAsync、promise 注册的回调
        - AsyncSeries: 异步串行钩子
        - AsyncParallel: 异步并行触发
    
    - 使用示例
      SyncHook
      SyncBailHook
      SyncWaterfullHook
      SyncLoopHook

      AsyncSeriesBailHook
  
  - 实现原理


## 题库
  1. 编写一个loader https://zhuanlan.zhihu.com/p/235553785
    1. 本质上是个函数, 接受文件内容, 输出新的内容，常用于资源模块转换
    2. 具体实现
      1. 在配置文件中的 module.rules 定义loader的描述对象, 包括目标文件匹配的正则、排除的路径、传给loader的options等
      2. 编写处理函数, 同步 loader 直接 return 即可, 异步loader需要调用 this.async(error, newContent) 输出转换后的内容
    3. 原则: 功能单一性、链式、无状态、尽可能异步
  
  2. 如何编写一个plugin
    1. 本质上是注册 webpack 构建时广播出来的生命周期钩子，通过webpack提供的api改变输出结果，以扩展功能
    2. 具体实现
      1. webpack会调用插件的 apply 方法并传递 compiler，可以通过该对象注册 生命周期钩子

  3. webpack的构建流程？（初始化-编译-输出）
    初始化参数 (基于shell传入的参数和配置文件)
    初始化编译: 初始化compiler, 加载 plugins, 开始编译
    确定入口: 根据 entry 配置确定入口文件
    编译模块: 从入口文件开始递归找到所有依赖的 module, 并用loader编译, 得到编译后的模块及其相互依赖关系
    组装代码块: 根据入口与模块的依赖关系组装出一个个包含一或多个模块的代码块, 加入到输出列表, 这是修改文件内容的最后机会
    输出: 根据输出列表写入文件系统

  4. 如何优化 Webpack 的构建速度？
    1. 精准匹配 (路径查询和文件名匹配): 缩小文件的搜索范围(通过test/include/exclude), 目的是 减少loader转化的数量 和 加快匹配速度
    2. 适当降低文件监听的频率(减小poll值, 其表示监听赫兹), 提高编译延时(增大aggregateTimeout的值)

  5. 文件监听原理？
    poll: 轮询文件最后修改的时间, 以得知文件变化
    aggregateTimeout: 文件变化后的延迟构建时间

  6. webpack 热更新原理？- webpack-dev-server 插件做了什么




  

  7、文件指纹是什么？
    指打包输出的文件的后缀名
    Hash: 和整个项目的构建相关, 只要项目文件有修改, 整个项目构建的 hash 值就会更改
    Chunkhash: 和 Webpack 打包的 chunk 有关, 不同的 entry 会生出不同的 chunkhash
    Contenthash: 根据文件内容来定义 hash, 文件内容不变, 则 contenthash 不变

  11、常见的 loader 和 plugin
    vue-loader, less-loader, css-loader, style-loader, babel-loader
    uglifyjs-webpack-plugin, HotModulePlugin, terser-webpack-plugin, CommonsChunkPlugin


```
webpack
  entry 入口配置, 支持 单文件路径、文件路径数组(将创建多个主入口)
    entry: { main: './main.js' }          // 单入口写法
    entry: './main.js'                    // 上一种的简写
    entry: ['./main1.js', './main2.js']   // 多入口写法
    entry: {                             // 对象语法, 扩展性最强
      app: './src/app.js',  // 应用脚本
      vendors: './src/vendors.js'  // 第三方库
    }
    entry: {                              // 多页应用, 为每个页面构建独立的依赖图
      pageOne: './src/pageOne/index.js',
      pageTwo: './src/pageTwo/index.js',
      pageThree: './src/pageThree/index.js'
    }

  plugins 插件, 比 loader 功能更加强大

  mode 模式 development(开发环境) / production(生产环境)

  modules 模块
    webpack 可以识别的模块依赖
      ES2015 import
      CommonJS require()
      AMD define 和 require()
      css/sass/less @import
      css中 url() 或 html 中 <img src=""> 的连接
  
  module resolution 模块解析
    解析规则
      绝对路径, 无需进一步解析
        import "/home/me/file";
        import "C:\\Users\\me\\file";
      相对路径, 使用import 或 require 的资源文件所在目录被认为上下文目录
        import "../src/file1";
        import "./file2";
      模块路径
        import "module";
        import "module/lib/file";
  
  manifest

  缓存

  热替换
```


## 与竟品比较
  - rollup

  - vite https://cn.vitejs.dev/guide/why.html#slow-updates
    https://mp.weixin.qq.com/s/DGFmV7WX7M5zKGAmlFdR4Q
    1. 开发环境采用浏览器 ESM(es module) 模块，省去了打包成 budnle 的时间，首次启动很快 (生产环境用rollup)
      - ESM介绍 https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Guide/Modules
      - 天然的代码分割，按需加载模块
    2. esbuild 预构建
      - 使用go语言编写的打包器，效率高
      - 解决依赖树大时，依赖请求数过多的问题
      - 消除多种模块引入方式的语法差异，以及模块的路径问题(ESM不支持 node_modules 的检索模式，只支持相对路径)
      - 提供更快的ts、jsx等转译功能
    3. 源码模块协商缓存，依赖模块强缓存，减少 ESM 通过网络请求加载模块的时间

  - parcel

  - webpack 的优势



