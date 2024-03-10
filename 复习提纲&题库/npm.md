npm 常用命令 https://blog.csdn.net/baoao1875/article/details/101642210
package.json https://mp.weixin.qq.com/s/dUXdwaH55hIl0C0pTIsUiw

## 版本管理 https://zhuanlan.zhihu.com/p/368454549
  - 版本号 (semver 约定) 1.0.0
    主版本
    小版本
    修订版本
  - 修饰符 [版本号]-beta https://cloud.tencent.com/developer/article/1521334
    alpha 内测版, 主要使用人员为开发与测试, 不推荐用于生产
    beta 公测版, 相对稳定的版本
    rc 预发版, 不再增加功能, 仅修复问题
    release 正式版, 最终发行版
    [版本号]-beta.x 修饰版本
  - 版本约束
    1.0.0   强制指定版本
    ^1.0.0
      不修改主版本, 即 [1.0.0 ~ 2.0.0)
      在有修饰符的场景下, 如 1.0.0-beta, 将锁定版本号, 仅允许 [1.0.0-beta, 1.0.0-stable]
      在有修饰版本的场景下, 如 1.0.0-beta.1, 将锁定版本号, 仅允许安装 [1.0.0-beta, 1.0.0-beta-x]
    ～1.0.0 
  - 查看npm包的历史版本
    npm view xxx version

## package-lock.json 锁文件
  生成策略
    若 package.json 被修改，且新版本与 lock 文件里的版本对照不符合 semver 规范，则更新 lock 文件
    semver 规范 

## 项目依赖
  https://blog.csdn.net/hujinyuan357/article/details/99621542
  https://zhuanlan.zhihu.com/p/390218026
  dependencies
  devDependencies
  peerDependencies: 行为取决于 宿主、模块的 dependencies，以及 npm 版本
    1. 宿主、模块的 dependencies 分别声明两个依赖版本，则宿主和模块的 node_modules 都进行安装
    2. 仅宿主的 dependencies 声明版本，则忽略模块的 peerDependencies，仅在宿主的 node_modules 中安装
    3. 宿主 dependencies 未声明依赖库的版本，则根据模块的 peerDependencies，只在模块的 node_modules 进行安装
    4、npm（3.x 版本之后，7.x 之前）、yarn 不会自动安装该配置下的依赖模块，即需要用 8.x 的 npm 安装


## npm link

## yalc https://mp.weixin.qq.com/s/QGzjE9QPzppttULsR3GuDg