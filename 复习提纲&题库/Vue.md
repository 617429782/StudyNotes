## MVVM 和 MVC
  1. MVC: 数据、视图、控制层(决策怎么使用数据)
  2. MVVM: 在数据和视图层之间加入了modelView层, 负责 数据和UI 的双向同步, 使得开发者不需要关注DOM操作、数据同步这些问题
  3. Vue 的 MVVM 实现是通过 defineProperty 定义数据的描述符, 在数据变化时完成 虚拟DOM 的更新, 然后渲染成真实的DOM

## 生命周期 每个组件都是一个Vue实例, 记为vm
  0. 准备阶段: 在Vue实例创建之前，通过 mixin、plugin 等对Vue构造函数进行一些扩展
    initGlobalAPI: 为Vue添加静态方法, 如 mixin、use(调用插件install)、extend、set、delete、nextTick 等
    xxxMixin: 以mixin的方式组织代码，主要有 
      initMixin: 
      stateMixin: 
      eventsMixin: 
      lifecycleMixin: 
      renderMixin: 
  1. 初始化阶段: 为vue实例添加必要的属性, 以及进行数据劫持
    1. initLifecycle: 为vm实例添加属性, 如 parent、children、_isMounted、_isDestroyed 等
    2. initEvents: 为vm添加 _events, 用于存放父组件为其绑定的事件
    3. initRender: 为vm添加 createElement 方法, 其作用是得到一个VNode
    -- beforeCreate --
    4. initInjections: provide/inject 类似react的context, 用于跨组件状态共享
    5. initState: 将option中的 props、methods、data(数据劫持)、computed、watch 添加到当前vm
    6. initProvide
    -- created --
  2. 挂载阶段: 主要工作是 模版编译, 挂载DOM
    1. 模版编译: 如果options中没有render, 则将 options.template 编译成 render 函数
      处理 options，得到 modules、directives
    -- beforeMount --
    2. 执行 vm._render 函数, 递归调用 createElement (子组件的创建也在这里), 拿到 组件树 对应的 虚拟DOM
    3. 执行 vm._updated 函数, 将 虚拟DOM 挂载到 真实DOM
    -- mounted --
  3. 更新阶段: 
    1. 在挂载阶段, 为每个组件都注册了一个 Watcher 实例, render 中用到的 state 都触发了 get, 将这个 watcher 收集到自己的依赖下
    2. 创建 Watcher 实例时, updateComponent 被作为 getter 而不是回调, 执行的时机是 watcher.run/evaluate
    -- beforeUpdate --
    3. 当 state 变动后, 统一在 nextTick 执行 watcher.run, 执行 vm._rander 拿到新的 虚拟DOM
    4. 执行 vm._update 进行 diff, 先判断能否复用, 依据主要是 key、tag, 不能复用删除, 能复用则进一步比对, patch Vnode 
    -- updated -- 
  4. 销毁阶段 beforeDestroy、destroyed
    -- beforeDestroy --
    1. 让父节点在 虚拟DOM树 中删除自己
    2. 清空挂在自己身上的 watcher
    3. 调用 浏览器接口删除 DOM
    -- destroyed -- 
    3. 清空挂在自己身上的 事件

## 双向绑定
  1. data -> view
    1. 初始化阶段对data进行数据劫持, 在挂载阶段触发 响应式的get函数 收集依赖, 在set时触发依赖, 通知组件重新
      每个响应式数据都闭包了一个 dep 对象, 用于收集和触发 watcher
      并不是每次 get 都会收集watcher, 仅在 new Watcher、watcher.update 时, 才会进行收集
    2. 因为 defineProperty 无法监听 对象属性 和 数组项 的变化, 对 数组原型方法进行劫持, 并提供了 $set、$delete 方法
  2. view -> data
    1. 模版编译阶段, 父组件绑定的事件都被注册到 本组件的事件中心 v._events
    2. 在 vm._update 生成真实DOM时, 调用 addEventListener 绑定DOM事件

## 虚拟DOM
  1. 本质是JS对象, 当数据改变的过程中, 修改JS对象 比 操作DOM 高效, 体现在
    1. nextTick 合并多次操作 (比如 多次修改样式、attr), 减少重绘回流
    2. 因为 diff, DOM操作被压缩到最小
    3. 现代浏览器中对于多次的DOM操作有合并优化, 上述的优点被削弱
  2. 主要属性有 key、tag、children、data: { attrs }、事件 等

## 事件
  1. 本质上是个发布订阅机制, 模版编译阶段收集父组件注册的事件到 vm._events
  2. 挂载阶段, 调用updateDOMListeners, 将事件通过 addEventListener/removeEventListener 注册到 vnode 对应的 DOM 上

## diff: 比对新旧虚拟DOM树 https://mp.weixin.qq.com/s/NurFhGUKcCRTJkk52P8_AQ
  1. 同层比较: 不考虑子节点的差异, 因为开发中很少遇到移动部分DOM到其他位置的情况, 也就没有复用的需求
  2. 优先复用: key相同直接复用(直接进入patch), tag相同有条件复用 (考虑是否注释节点、input标签等情况)
  3. 四指针-双端比较(优化): 
    旧头 - 新头
    旧尾 - 新尾
    旧头 - 新尾
    旧尾 - 新头
    只要有其中一种规则判断可以复用, 则该节点进入 patch 阶段, 指针顺移
    节点移动: 当双端比较判断到新旧节点相同, 但是位置不同(一个头一个尾), 则需要移动节点
  4. 常规比较: 从 新列表依次拿出节点 与 旧节点列表 比对
    新节点 在旧列表中存在, patch, 并调用 insertBefore 将该节点插入到 旧头 之前
    新节点 在旧列表中不存在, 调用 createElement 创建
    新列表 遍历完旧列表头尾指针中的所有节点, 调用 removeChild 删除

## 异步渲染 / nextTick: 延迟render, 合并渲染请求
  1. 调用 nextTick 可以将一个 函数 存入 缓存队列, 在本次宏任务完成后, 统一在 微任务阶段 执行
  2. 短时间内多次重渲染会导致性能浪费, vue将更新任务存入缓存队列, 通过watcher.id剔除重复任务
  3. state 变更触发 render 重新生成 虚拟DOM 的过程, 以及后续 patch 到 DOM树 的过程, 本质上都是宏任务
  4. nextTick 利用 微任务 延迟 render消费, 等待 当前脚本 所有的 render任务 产生完毕
  5. 应用场景: 在更新任务完成后操作DOM, 比如 让 v-if 变为 true 的输入框获取焦点, 就需要在 修改 v-if 后, 调用 nextTick 添加回调

## computed / watch
  1. computed 具有缓存特性, 其依赖set时不立即触发回调, 而是在下次 computed 被使用时触发
  2. computed 采用的是 lazy-watcher, 把值缓存在 watcher.value, 依赖的数据变化时只更新 value, 而不会触发回调

## 指令
  1. 指令的执行流程
    1. 通过 options 注册声明指令, 与内置指令合并, 得到最终的 directives
    2. 模版编译阶段, 解析DOM属性时, 会使用正则( v- 开头)找出 指令属性 (v-on、@ 等也是在这一阶段收集), 添加到 el.directives 上
    3. 挂载/更新阶段, 触发生命周期函数, 调用 指令注册的钩子

  2. 指令的作用本质是通过钩子函数, 得到操作 当前虚拟DOM节点 的能力
    bind：只调用一次, 指令第一次绑定到元素时调用。在这里可以进行一次性的初始化设置。
    inserted：被绑定元素插入父节点时调用 (仅保证父节点存在, 但不一定已被插入文档中)。
    update：所在组件的 VNode 更新时调用
    componentUpdated：指令所在组件的 VNode 及其子 VNode 全部更新后调用。
    unbind：只调用一次, 指令与元素解绑时调用。
  
  3. 题库
    1. v-if 与 v-show 的区别
      v-if 会触发组件完整的生命周期, 对其进行销毁和重建
      v-show 仅修改 DOM 的 display 属性, 不进行组件的销毁

    2. 为什么 v-for 和 v-if 不能一起使用
      v-for 的执行时机在 v-if 之前, 这可能会导致 v-for 处理了 v-if 为 false 的节点, 造成浪费

    3. 为什么不用 index 作为 key
      Vue在diff新旧虚拟DOM时有 优先复用原则, 即不进入 patch 的过程
      如果使用 index, 假如在列表最前面插入一个新节点, 原本可以被复用的后续节点将全部进入 patch

## 组件通信
  1. prop + $emit
    每个vm初始化阶段都创建了 _events 存储父组件注册的事件
  2. provide/inject 依赖注入, 类似于 react 的 context, 可跨组件传递
    1. initInjections
      向上递归找 父组件的 _provided 属性, 添加到自身 vm 下, 与 data 不同, 这一步拿到的属性 不做响应式劫持, 除非 父元素中定义的对象本身就是响应式
    2. initProvide
      将 provide 传入的 对象 挂在自身 vm 下, 如果传入的是函数类型, 则挂载 函数的执行结果(函数类型可以通过this拿到vm下的数据, 如 data、computed)
  3. vuex
  4. Event 事件中心

## 插槽 / 作用域插槽
  1. 插槽 理解为 子组件 为 父组件 预留的占位符
    子组件成解析 虚拟DOM 时, 会从 父组件 拿到对应的内容 (默认或具名)

  2. 作用域插槽 在从 父组件 拿内容时会携带参数, 作为内容部分的 props
    因此: 作用域插槽下, 父组件提供的 内容部分, 可以访问 子组件的数据

  3. vue@2.6 中 v-slot 代替了 原先的 slot 和 slot-scoped 写法 (可以认为所有的插槽都可以通过在 <slot /> 上加属性变为作用域插槽)
    v-slot 只能用于 <template>, 除非 只有默认插槽  

## createElement(tag, [ data(Object), children(String|Array) ]) 函数

## keep-alive https://segmentfault.com/a/1190000011978825
  keep-alive 是一类特殊的抽象组件, 用于保持不活跃组件的状态, 减少组件的销毁重建带来的不必要的 render
  其将 组件实例 缓存在自身 cache 字段中, 在 render 时基于传入 include 和 exclude 属性判断是否在返回的 vNode 中带上缓存的 组件实例

## 异步加载组件
  配合webpack的代码分割, 在组件定义传入回调的时候传入一个promise, 并利用require/import告知webpack需要进行代码分割
  以下几种方式都可以
    Vue.component('async-webpack-example', (resolve) => { require(['./my-async-component'], resolve) })
    Vue.component('async-webpack-example', () => import('./my-async-component'))
    new Vue({
      components: {
        'my-component': () => import('./my-async-component')
      }
    })

## Vue.mixin
  代码按逻辑组合的方式, 本质上是根据 merge 规则, 将多个对象合并成一个大对象

## 异常处理 https://mp.weixin.qq.com/s/hgD3nOwLAh2fiEO2HWcqIA
  Vue.config.errorHandler: 全局错误处理
    会被放在 errorCaptured 调用队列的顶层
  errorCaptured 钩子: 冒泡 的组件错误处理
    生命周期函数:
      callHook 触发钩子函数时，会用 try/catch 对函数进行包装，捕获同步函数的异常
      如果是函数返回值是 promise，还会调用 promise.catch 进行捕获
      捕获到异常后，逐级访问 $parent 拿到祖先组件的 errorCaptured 钩子，直到其中有钩子返回 false，或到根组件
    事件函数
      与生命周期函数类似，事件函数也会被调用同样的方法进行包装
      区别在于【事件函数的包装时机】是在 将 vnode 上的 on 属性绑定为 DOM事件时

## Vue3 新特性
  1. Proxy 重写数据劫持
  2. composition Api: 类似于 hooks, 提供了新的逻辑组合方式, 取代 mixin
  3. 编译阶段进行静态标记, 不参与diff
  4. 引入 ts 取代之前的 flow
  5. 自定义渲染, fragment, teleport, suspense

## vue-router https://mp.weixin.qq.com/s/h6rKfXINQKB0499ZK5beTQ
  1. html history 对象: 以栈维护访问过的页面
    1. api 列表
      1. back() 后退  /  forward() 前进  /  go(step) 前进/后退 step 个页面
      2. pushState(state, title, url)  /  replaceState(state, title, url)
        分别在 栈顶 添加/替换 一条记录
        两者都不能改变域名, 仅修改域名后的路径, 即 location.pathname
    2. 真实的访问记录
      对于用户真实访问过的页面, back/forward/go(step) 可以跳转到对应页面并重新加载
    3. 手动添加的的访问记录
      对于 pushState 手动添加的记录, back/forward/go(step) 不会改变当前页面, 仅改变 url
    4. popstate 事件
      不管记录来源是真实的, 还是手动的, 都会触发
  2. hash 属性: 用于指导浏览器滚动到对应位置, 对服务端无效, 其改变不会刷新页面

  3. 路由传参

  4. 路由懒加载

## 题库
  - 简述 MVVM

  - 响应式原理

  - 双向绑定原理

  - 异步渲染 / nextTick

  - vue 组件的生命周期

  - 父子组件生命周期的调用顺序
    父 created -> 子 created -> 子 monuted -> 父 mounted
    父 created async 修饰后: 
    父 created 未执行完 -> 子 created -> 子 monuted -> 父 created -> 父 mounted
    可能会导致 子组件取不到父组件的值 https://www.jianshu.com/p/7a56fca39591

  - computed / watch 的特定和实现原理

  - vue 的事件绑定原理

  - 为什么data是函数
    组件被定义时可以视为一个类, 可能用于创建多个实例, 所有的实例 data 会挂在组件原型属性 $data 下, 若直接传入对象(引用类型)会导致组件之间的相互覆盖, 用函数可以保证传入的是一个新的对象
    
  - 为什么传统diff要 O(n^3), vue的diff是 O(n)
    O(n^3) = 判断相等(遍历新树n * 遍历旧树n) * 移动位置n
    vue 同层比较, 边判断边移动, 理想情况下 O(n), 坏的情况下 O(n^2)
