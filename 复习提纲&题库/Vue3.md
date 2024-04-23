## Vue3 新特性
1. 响应式原理:
  ```
  Vue2 是对原数据的深层改造, 对于引用类型需要额外的处理
  Vue3 是为原数据生成一个代理对象，解构这类行为会绕过代理对象而导致响应式失效
  ```
2. 静态分析
  ```
  静态提升: 提升静态内容到 render 范围外进行复用，直接通过 innerHTML 挂载，而不是 patch
  更新类型标记: 分析元素的动态部分，比如 :class, 在 createElementVNode 时传入 第四个参数，标记更新的类型，这里用了位运算来合并多种标记
  树结构打平: 对于区块节点(无 if、for)在编译时忽视树形结构，将后代中的动态节点以数组方式存放，这在DOM层级深的情况下 diff 效率更高
  ```
3. 组合式 API
  ```
  提供了另一种逻辑复用选项，多个 Mixin 容易产生相互依赖和冲突，存在隐性的耦合而导致维护难度升高
  组合式 API 提供的 hooks 能力，避免同一个逻辑关注点被分散在多个 options 配置中，使得逻辑更加内聚，更易维护
  更有利 TS 类型推导
  ```

## 响应式原理 https://mp.weixin.qq.com/s/F2yYqXE_xTHl0d8j03I-UQ
- 用 proxy 重构响应式的背景 => defineProperty 操作 引用类型 的局限性
- 语法: relative 和 ref
  - relative: 适用于引用类型，深层处理响应式，等同于 Vue.observable()
  - ref: 适用于基本类型，读写时需要通过 .value
- 新响应式的弊端
  - 解构会丢失响应式特性
  - 需要用 .value 来进行值的操作
- Proxy 响应式的工作原理
  - 实现思路
    ```
    function reactive(obj) {
      return new Proxy(obj, {
        get(target, key) {
          track(target, key)
          return target[key]
        },
        set(target, key, value) {
          target[key] = value
          trigger(target, key)
        }
      })
    }

    function ref(value) {
      const refObject = {
        get value() {
          track(refObject, 'value')
          return value
        },
        set value(newValue) {
          value = newValue
          trigger(refObject, 'value')
        }
      }
      return refObject
    }
    ```
  - 代理【对象类型】的 [13种基础操作](../ES6笔记/proxy.md)
  - 对于【基本类型】需要通过 { value: x } 进行包装后代理, 原因是 JS 不提供对局部变量读写操作的追踪，但可以追踪对象属性的读写
    ```
      // 基本类型的 代理对象，拥有对 value 属性的读写操作
      class RefImpl {
        public _value;
        public dep = new Set; // 依赖收集
        public __v_isRef = true; // 是ref的标识
        // rawValue 传递进来的值
        constructor(public rawValue, public _shallow) {
          // 1、判断如果是对象 使用reactive将对象转为响应式的
          // 浅ref不需要再次代理
          this._value = _shallow ? rawValue : toReactive(rawValue);
        }
        get value() {
          // 取值的时候依赖收集
          trackEffects(this.dep)
          return this._value;
        }
        set value(newVal) {
          if (newVal !== this.rawValue) {
            // 2、set的值不等于初始值 判断新值是否是对象 进行赋值
            this._value = this._shallow ? newVal : toReactive(newVal);
            // 赋值完 将初始值变为本次的
            this.rawValue = newVal
            triggerEffects(this.dep)
          }
        }
      }
    ```
  - 为什么解构赋值会导致响应式失效？
    因为 proxy 本质上是对【对象】的操作代理，如 a = { b: "" }，是对 a 的读写代理
    解构赋值 const { b } = a 相当于绕过了 a，直接操作 b，也就无法触发代理，const c = a.b 也是同理

## Vue3 渲染流程 https://juejin.cn/post/7206184389298241593
  源码 https://github.com/vuejs/core/blob/main/packages/runtime-core/src/vnode.ts
  subTree 是一个 vnode，记录了 组件的 tempalte 结构

## v-model 的变化
- Vue2: 模版编译阶段，编译成 value + input
- Vue3: 融合了 Vue2 中的 .sync 受控属性，编译成 propName + update:propName
  - 对于原生 DOM，即 input、select、textarea 仍然编译为 value + input
  - 对于组件，编译为 modelValue + update:modelValue
  - 对于有修饰符的配置，如 (v-model:title.capitalize), 将生成 title 和 titleModifiers 两个 prop

## 自定义渲染 (render、h)[https://cn.vuejs.org/guide/extras/render-function.html]
- 与 vue2 createElement 的差异
  todo: 参数差异

## 组合式 API vs 选项式 API
- 组合式: 支持 <srcipt setup /> 和 setup(props, ctx) 两种用法，本文内称为 声明式 和 函数式
  - 声明式: 
    - 顶层声明的变量、函数以及 import 的内容可以直接供模版使用
    - 对外暴露的
  - 函数式:
    - 供模版和对外暴露的内容，需要在 setup 函数中 return
    - 可以与选项式共存，return 的内容可以在选项式中通过 this 访问，反之则不行
    - context 可设置 attrs、slots、emit、expose
    - 可以 return 一个函数，将作为 render 使用，此时对外暴露的内容可以通过 expose 来设置
  - 生命周期钩子 https://cn.vuejs.org/api/options-lifecycle.html#beforecreate
    - 常用: onBeforeMount、onMounted、onBeforeUpdat、onUpdated、onBeforeUnmount、onUnmounted
    - 特定场景: onActivated、onDeactivated (keep-alive)、onErrorCaptured (异常捕获)

- 选项式: 与 vue2 基本相同
  - 状态选项
    data
    props
    computed
    methods
    watch
    emits: 声明接受的自定义事件
    expose: 定义对外暴露的属性（比如被父组件通过 ref 引用时可拿到的属性）

  - 渲染选项
    template / render 可选
    compilerOptions 模版编译选项，仅在完整版 Vue 生效
    slots 用于辅助 render 函数，声明将使用的插槽（3.3+）

  - 生命周期
    基础：同 Vue
    renderTracked、renderTriggered: 调试模式特有
    serverPrefetch: ssr 特有

  - 组合选项
    依赖注入 provide / inject
    mixin、extends
    inheritAttrs 是否启用 attribute 透传行为
    directives
    componets

  - this 对象: 相比 vue2 删减了一些原型属性，比如 $set、$on 等
    $data
    $props
    $el
    $options
    $parent
    $root​
    $slots
    $refs
    $attrs
    $watch()
    $emit()
    $forceUpdate()
    $nextTrick

## 选项式 API

## watchEffect 和 watch https://vue3js.cn/vue-composition-api/#watcheffect
- watch 与 $watch 行为相同
- watchEffect

## 响应式调试
- 组件渲染调试
  renderTracked: 查看哪些依赖正在被使用
  renderTriggered: 查看哪个依赖正在触发更新
- 计算属性、侦听器 调试
  onTrack 将在响应属性或引用作为依赖项被跟踪时被调用
  onTrigger 将在侦听器回调被依赖项的变更触发时被调用