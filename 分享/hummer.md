1. 基于 astrory 添加 hummer plugin，其中包括
  中间件: 通过 ctx.setState 挂载 hummer 的配置，主要有 appKey、上传的url 等，主要是给 后续的 模版渲染使用
  渲染模版: 
    ```
      <script
        onerror="_cdnFallback(this)"
        src="//b.yzcdn.cn/hummer/hummer-browser/index.vue-1.3.1.js"
        crossorigin="anonymous"
      ></script> 
    ```
    1. script crossorigin:  https://juejin.cn/post/6969825311361859598
      用于跨域资源内报错，拿到具体的报错内容
      原理是 cors 校验，即匹配 Access-Control-Allow-Origin

    2. index.vue-1.3.1.js: 定义 YouZanHummer 函数
      install
        window.onerror
        window.onunhandledrejection
        XHR
        requestAnimationFrame

        Vue.config.errorHandler

      劫持 addEventListener、removeEventListener，自定义错误对象

      性能上报 window.performance https://zhuanlan.zhihu.com/p/30329705



    3. YouZanHummer.Hummer.init(config)


## ctx.setState、ctx.setGlobal  
  setState: 编译时属性，给 /view 下 模版编译时 使用
  setGlobal: 运行时属性，挂载到 windo._global 下

## script 属性 https://developer.mozilla.org/zh-CN/docs/Web/HTML/Element/script
  async,
  defer,
  integrity,
  crossorigin
 



## Hummer 实例
  1. 捕获器