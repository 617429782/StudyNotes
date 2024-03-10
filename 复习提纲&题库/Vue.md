## MVVM 和 MVC
  1. MVC: 数据、视图、控制层(决策怎么使用数据)
  2. MVVM: 在数据和视图层之间加入了modelView层, 负责 数据和UI 的双向同步, 使得开发者不需要关注DOM操作、数据同步这些问题
  3. Vue 的 MVVM 实现是通过 defineProperty 定义数据的描述符, 在数据变化时完成 虚拟DOM 的更新, 然后渲染成真实的DOM

## 生命周期
  0. 准备阶段: 在Vue实例创建之前，通过 mixin、plugin 等对Vue构造函数进行一些扩展
    initGlobalAPI: 为Vue添加静态方法, 如 mixin、use(调用插件install)、extends、set、delete、nextTick 等
    xxxMixin: 以mixin的方式组织代码，主要有 
      initMixin: 
      stateMixin: 
      eventsMixin: 
      lifecycleMixin: 
      renderMixin: 挂载 render辅助函数、$nextTick、_render 等原型方法
  1. 初始化阶段(_init): 为vue实例添加必要的属性, 以及进行数据劫持
    1. initLifecycle: 为vm实例添加属性, 如 parent、children、_isMounted、_isDestroyed 等
    2. initEvents: 为vm添加 _events, 用于存放父组件为其绑定的事件
    3. initRender: 为vm添加 createElement 方法, 其作用是得到一个VNode
    -- beforeCreate --
    4. initInjections: provide/inject 依赖注入，类似react的context, 用于跨组件状态共享
    5. initState: 将option中的 props、methods、data(数据劫持)、computed、watch 添加到当前vm
    6. initProvide
    -- created --
  2. 挂载阶段($mount): 主要工作是 模版编译, 挂载DOM
    1. 模版编译: 如果 options 中没有 render, 则调用 parse 将 options.template 编译成 render 函数
      过程简述: template (parse 解析) => ASTElement (optimize 标记优化) => staticRoot (codegen 代码生成) => render 函数
        parse 得到 ast
        render(createElement) 得到 vnode
        update & patch 得到 DOM
      处理 options，得到 modules、directives
    -- beforeMount --
    2. 为 vm 创建一个 Watcher 实例，同时将 updateComponent 被作为 getter 而不是回调传入, 相当于立即调用，此时开启了订阅窗口，会对 updateComponent 过程中用到的劫持数据进行订阅
      (执行了 _render + _update 就相当于用了 data，也就触发了 get，使这个 Watcher 成为了 data 对应 dep 的订阅者，当 data 变化就触发了 set，会因为想要读取 Watcher.value 而重新触发 getter，也就重新进行了 updateComponent)
    -- updateComponent --
    3. 执行 vm._render 函数, 递归调用 createElement (子组件的创建也在这里), 拿到 组件树 对应的 虚拟DOM
    4. 执行 vm._update 函数, 进行 patch，将 虚拟DOM 挂载到 真实DOM
    -- mounted --
  3. 更新阶段: 
    1. 在挂载阶段, 为每个组件都注册了一个 Watcher 实例, render 中用到的 state 都触发了 get, 将这个 watcher 收集到自己的依赖下
    2. 创建 Watcher 实例时, updateComponent 被作为 getter 而不是回调, 相当于被立即调用
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

## 模版编译
  0. 执行时机：$mount 阶段，入口函数 compileToFunctions
    调试方式：定义一个带 template 的组件，绕开 vue-loader，否则会在打包阶段被优化成 render 函数以提升性能
  1. parse 解析 => ASTElement
    parseHTML: 通过不断得正则 match，拿到元素的 开始、结束、字符、注释 部分，组装成一个 ast 对象
      <div>
        <input :key="idx" v-for="(col, idx) in arr" v-model="col.name" />
      </div>

      ast: {
        attrsList: []
        attrsMap: {}
        children: [
          alias: "col"
          attrsList: [{name: 'v-model', value: 'col.name', start: 54, end: 72}]
          attrsMap: {
            :key: 'idx', 
            v-for: '(col, idx) in arr', 
            v-model: 'col.name'
          }
          children: []
          directives: [
            arg: null
            end: 72
            isDynamicArg: false
            modifiers: undefined
            name: "model"
            rawName: "v-model"
            start: 54
            value: "col.name"
          ]
          end: 75
          for: "arr"
          hasBindings: true
          iterator1: "idx"
          key: "idx"
          parent: {type: 1, tag: 'div', attrsList: Array(0), attrsMap: {…}, rawAttrsMap: {…}, …}
          plain: false
          rawAttrsMap: {:key: {name: ':key', value: 'idx', start: 17, end: 27}, v-for: {…}, v-model: {…}}
          start: 10
          tag: "input"
          type: 1
        ]
        end: 84
        parent: undefined
        plain: true
        rawAttrsMap: {}
        start: 0
        tag: "div"
        type: 1
      }

  2. optimize 标记优化 => 为 astNode 加 static 属性, 标明一些节点不参与 patch，有利 update 性能
    被标为静态的条件有
      文本节点 astNode.type = 3, 或
      无 bind、无 if、for、且所有子节点都是静态节点的 html标签

  3. codegen 代码生成 => 将 ast 转为 render 函数
    以下分支互斥(忽略冷门分支)
      el.for && !el.forProcessed => genFor
      el.if && !el.ifProcessed => genIf
      el.tag === 'template' && !el.slotTarget => genChildren
      el.tag === 'slot' => genSlot
      component or element
        el.component => genComponent
        genChildren

    render 函数示例, render 函数是用 with(this) 包裹的，因此可以直接访问 data 中的变量
      _c(
        "div",
        _l(arr, function (col, idx) {
          return _c("input", {
            directives: [
              {
                name: "model",
                rawName: "v-model",
                value: col.name,
                expression: "col.name",
              },
            ],
            key: idx,
            domProps: { value: col.name },
            on: {
              input: function ($event) {
                if ($event.target.composing) return;
                $set(col, "name", $event.target.value);
              },
            },
          });
        }),
        0
      );
    
    内置函数说明
      _c = createElement;         // 创建 vnode
      _o = markOnce;
      _n = toNumber;
      _s = toString;
      _l = renderList;            // for 循环
      _t = renderSlot;            // 插槽
      _q = looseEqual;
      _i = looseIndexOf;
      _m = renderStatic;
      _f = resolveFilter;
      _k = checkKeyCodes;
      _b = bindObjectProps;
      _v = createTextVNode;
      _e = createEmptyVNode;
      _u = resolveScopedSlots;
      _g = bindObjectListeners;
      _d = bindDynamicKeys;
      _p = prependModifier;

## 双向绑定
  1. data -> view
    1. 初始化阶段对data进行数据劫持, 在挂载阶段触发 响应式的get函数 收集依赖, 在set时触发依赖, 通知组件重新
      每个响应式数据都闭包了一个 dep 对象, 用于收集和触发 watcher
      并不是每次 get 都会收集watcher, 仅在 new Watcher、watcher.update 时, 才会进行收集
    2. 因为 defineProperty 无法监听 对象属性 和 数组项 的变化, 对 数组原型方法进行劫持, 并提供了 $set、$delete 方法
  2. view -> data
    1. 模版编译阶段, 父组件绑定的事件都被注册到 本组件的事件中心 v._events
    2. 在 vm._update 生成真实DOM时, 调用 addEventListener 绑定DOM事件
  
## 响应式原理
- vue2 defineProperty get/set
  - 基本原理: 观察者模式
  - 局限性: 引用类型的操作不会触发 set 方法
    - 数组子项变化 => 劫持 push、pop 等方法
    - 对象的属性变化 => $set、$delete
  - observable(target) vs $set(target, path)
    - $set 对非响应式对象使用时，仅赋值，对响应式对象使用时，会调用 defineReactive 为其增加一个响应式属性
    - observable 底层用的也是 defineReactive，可以将非响应式对象转为响应式

- vue3 proxy

## 虚拟DOM
- 概念
  - 本质是JS对象, 当数据改变的过程中, 修改JS对象 比 操作DOM 高效, 体现在
    nextTick 合并多次操作 (比如 多次修改样式、attr), 减少重绘回流
    因为 diff, DOM操作被压缩到最小
    现代浏览器中对于多次的DOM操作有合并优化, 上述的优点被削弱
  - 主要属性有 key、tag、children、data: { attrs }、事件 等

- $vnode 和 _vnode 的差异 https://juejin.cn/post/6844904135209271303
  vm._vnode 是上一次 render 的结果，在 patch 时与本次 update 接收的 vnode 进行 diff 比对
  ```
    Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {
      const prevVnode = vm._vnode
      vm._vnode = vnode
      vm.__patch__(prevVnode, vnode)
    }
  ```
  vm.$vnode 是 componentInstance 所对应的 vnode, 为 vnode 创建组件时，vnode 自身会作为 parentVnode 存在组件 options 中，体现 组件由 vnode 创建
  ```
    function createComponentInstanceForVnode (
      vnode: any,
      parent: any
    ): Component {
      const options: InternalComponentOptions = {
        _isComponent: true,
        _parentVnode: vnode,
        parent
      }
      return new vnode.componentOptions.Ctor(options)
    }
  ```

## $createElement(tag, [ data(VNodeData), children(String|Array) ]): vnode
- Class VNode
  tag: string | void;
  data: VNodeData | void;
  children: ?Array<VNode>;
  text: string | void;
  elm: Node | void;
  ns: string | void;
  context: Component | void; // rendered in this component's scope
  key: string | number | void;
  componentOptions: VNodeComponentOptions | void;
  componentInstance: Component | void; // component instance
  parent: VNode | void; // component placeholder node

  // strictly internal
  raw: boolean; // contains raw HTML? (server only)
  isStatic: boolean; // hoisted static node
  isRootInsert: boolean; // necessary for enter transition check
  isComment: boolean; // empty comment placeholder?
  isCloned: boolean; // is a cloned node?
  isOnce: boolean; // is a v-once node?
  asyncFactory: Function | void; // async component factory function
  asyncMeta: Object | void;
  isAsyncPlaceholder: boolean;
  ssrContext: Object | void;
  fnContext: Component | void; // real context vm for functional nodes
  fnOptions: ?ComponentOptions; // for SSR caching
  devtoolsMeta: ?Object; // used to store functional render context for devtools
  fnScopeId: ?string; // functional scope id support

- VNodeData:
  key?: string | number;
  slot?: string;
  scopedSlots?: { [key: string]: ScopedSlot | undefined };
  ref?: string;
  refInFor?: boolean;
  tag?: string;
  staticClass?: string;
  class?: any;
  staticStyle?: { [key: string]: any };
  style?: string | object[] | object;
  props?: { [key: string]: any };
  attrs?: { [key: string]: any };
  domProps?: { [key: string]: any };
  hook?: { [key: string]: Function };
  on?: { [key: string]: Function | Function[] };
  nativeOn?: { [key: string]: Function | Function[] };
  transition?: object;
  show?: boolean;
  inlineTemplate?: {
    render: Function;
    staticRenderFns: Function[];
  };
  directives?: VNodeDirective[];
  keepAlive?: boolean;

## render 函数
- 作用: 递归调用 createElement，构建 vnode 树
- 原理: 
  render 函数暴露 $createElement 给用户，内部执行: render.call(vm._renderProxy, vm.$createElement), 生产环境下 vm._renderProxy 即 vm
  initRender 对 $createElement 做了包装，传入了 vm 作为当前 vnode 的 context

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
    
    4. v-model 的作用原理
      模版编译阶段的指令解析 <input v-model="parentInput" />，得到 input 事件、value 属性
        h('input', {
          data: {
            on: {
              input: e => {
                this.parentInput = e.target.value
                this.$emit("input", e.target.value)
              },
            },
            domProps: {
              value: this.parentInput
            }
          }
        })

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
  - 概念
    1. 插槽 理解为 子组件 为 父组件 预留的占位符
      子组件在解析 虚拟DOM 时, 会从 父组件 拿到对应的内容 (默认或具名)
    2. 作用域插槽 在从 父组件 拿内容时会携带参数, 作为内容部分的 props
      因此: 作用域插槽下, 父组件提供的 内容部分, 可以访问 子组件的数据
    3. vue@2.6 中 v-slot 代替了 原先的 slot 和 slot-scoped 写法 (可以认为所有的插槽都可以通过在 <slot /> 上加属性变为作用域插槽)
      v-slot 只能用于 <template>, 除非 只有默认插槽

  - 用法
    1. 匿名插槽: 等同于名为 default 的具名插槽
      Child: <div><slot /></div>
      Parent: <Child>匿名插槽</Child>
    2. 具名插槽: 需要配合 template 使用
      Child: <div><slot name="header" /></div>
      Parent: <Child><template v-slot:header>具名插槽</template></Child> 或 <Child><template #header>具名插槽</template></Child>
    3. 动态插槽名: 使用 动态指令参数 []
      Parent: <Child><template v-slot:[slotName]>具名插槽</template></Child> 或 <Child><template #[slotName]>具名插槽</template></Child>
    4. 作用域插槽
      Child: <div><slot :row="data" /></div>
      Parent: <Child v-slot="slotProps">{{ slotProps.row }}</Child>
      Child: <div><slot name="header" :row="data" /></div>
      Parent: <Child #header="slotProps">{{ slotProps.row }}</Child>

  - 原理
    1. $slots、$scopedSlots 的作用

## keep-alive https://segmentfault.com/a/1190000011978825
- 概念: keep-alive 是一类特殊的抽象组件, 用于保持不活跃组件的状态, 减少组件的销毁重建带来的不必要的 render
  其将 componentInstance 缓存在 this.cache 字段中, 在 render 时基于传入 include 和 exclude 判断, 是否将 vnode.componentInstance 改为 this.cache[key].componentInstance
  在更新队列 flushSchedulerQueue 执行时，会依次触发 activatedChildren 中的 activated 钩子，流程与 updated 相似

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

## Vue.mixin & extends 两者的合并策略一致
  本质上是组件 options 的合并
  - data: 深度合并，当前组件优先
  - 生命周期: 合并成数组，执行顺序为 前 => 后 => 组件定义
  - methods、components、directives：同名替换，当前组件优先

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
    父 beforeDestroy -> 子 beforeDestroy -> 子 destroyed -> 父 destroyed
    可能会导致 子组件取不到父组件的值 https://www.jianshu.com/p/7a56fca39591

  - computed / watch 的特定和实现原理

  - vue 的事件绑定原理

  - 为什么data是函数
    组件被定义时可以视为一个类, 可能用于创建多个实例, 所有的实例 data 会挂在组件原型属性 $data 下, 若直接传入对象(引用类型)会导致组件之间的相互覆盖, 用函数可以保证传入的是一个新的对象
    
  - 为什么传统diff要 O(n^3), vue的diff是 O(n)
    O(n^3) = 判断相等(遍历新树n * 遍历旧树n) * 移动位置n
    vue 同层比较, 边判断边移动, 理想情况下 O(n), 坏的情况下 O(n^2)
