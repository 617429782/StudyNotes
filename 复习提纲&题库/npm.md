npm 常用命令 https://blog.csdn.net/baoao1875/article/details/101642210

## package.json  
  - bin: 可执行文件列表  
    常用于 cli 工具, 会在被 install 后添加到 node_modules/.bin 和 PATH 文件中(全局安装时) 
     
  - main / module / browser: 各模块下的入口文件，**重点掌握 ESM vs CJS**  
    node 默认的 commonJS 规范要求使用 require + module.exports  
    ESM 规范使用 import + export，需要在 package.json 中声明 type: module  
    [相比 CJS，ESM](https://juejin.cn/post/6844903861166014478) 具有 引用输出、只读引入、import 提升 等特性，更有利 静态检查和tree-sharking  
    babel 通过修改赋值操作、同步修改 exports 值来模拟 只读引入和引用输出 的行为，但不能模拟 import 提升  

  - dependencies: **重点掌握 常用三种依赖的区别**  
    dependencies: 用于生产环境的包，如 lodash  
    devDependencies: 非生产环境的包，如 ellint、Typescript，即 --production 或 NODE_ENV 为 production 时不安装  
    peerDependencies: 插件要求宿主安装的依赖，插件自身的 node_module 不安装  
    ~~optionalDependencies: 可选依赖，在安装异常时仍继续执行，不常用~~  
    ~~bundledDependencies: []: npm pack 的时候用到，不常用~~  
    参考资料
      https://blog.csdn.net/hujinyuan357/article/details/99621542

  - files:   
    声明被 install 后，哪些文件在出现在宿主的 node_modules 里（通常不需要配置，打包环境基本上就已经处理好）

  - script  
    pre / post 在命令执行前后处理，如 { "start": "xxx", "prestart": "" }

  - version：版本管理  
    - 版本号 [semver 约定](https://semver.org/lang/zh-CN/)   
      主版本: 不兼容的改动  
      小版本: 向下兼容的功能  
      修订版本: 向下兼容的问题修复  
    - [修饰符](https://zhuanlan.zhihu.com/p/368454549) [版本号]-[修饰符]  
      alpha 内测版, 主要使用人员为开发与测试, 不推荐用于生产  
      beta 公测版, 相对稳定的版本  
      rc 预发版, 不再增加功能, 仅修复问题  
      release 正式版, 最终发行版  
      [版本号]-beta.x 修饰版本  
    - 版本约束  
      1.0.0  强制指定版本  
      ^1.0.0  
      > 不修改主版本, 即 [1.0.0 ~ 2.0.0)  
      > 在有修饰符的场景下, 如 1.0.0-beta, 将锁定版本号, 仅允许 [1.0.0-beta, 1.0.0-stable]  
      > 在有修饰版本的场景下, 如 1.0.0-beta.1, 将锁定版本号, 仅允许安装 [1.0.0-beta, 1.0.0-beta-x]  
      ～1.0.0   
    - 查看 npm 包的历史版本: npm view xxx version  
    
  - 参考资料 https://mp.weixin.qq.com/s/VEByRtbk9umGsS7od2PPOg

## package-lock.json 锁文件
  生成策略
    若 package.json 被修改，且新版本与 lock 文件里的版本对照不符合 semver 规范，则更新 lock 文件
    semver 规范 

## npm link

## yalc https://mp.weixin.qq.com/s/QGzjE9QPzppttULsR3GuDg