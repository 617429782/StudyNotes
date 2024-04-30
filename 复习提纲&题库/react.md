## [react 架构](https://react.iamkasong.com/preparation/newConstructure.html#react16-%E6%9E%B6%E6%9E%84)
1. [15 版本: 协调器 + 渲染器]()
  - [协调器](): 在状态变更时, 自根组件起, 调用组件 render 方法得到 虚拟DOM, 并与上一次的结果进行 diff 比对, 通知 渲染器 重新渲染差异部分的 虚拟DOM。
  - [渲染器](): 接收协调器的通知，进行虚拟DOM向目标对象的转化, ReactDOM 是最常见的渲染器, 此外还有 ReactNative、ReactArt 等。
  - 协调器和渲染器交替工作, 这是一个递归的过程。
2. [16 版本: 调度器 + 协调器 + 渲染器]()
  - [调度器 Scheduler](): 模拟了 requestIdleCallback 的功能，并加入了任务优先级设置的功能。
  - [协调器](): 由递归变为了链表遍历, 在发现虚拟DOM差异时, 不立即通知渲染器, 而是先为更新打上标记, 等待所有组件都完成打标后再统一发送，这个过程可以中断。
  - [渲染器](): 根据协调过程中打的标，进行对应的更新操作，这个过程不会中断。

## 调度器
- 任务优先级: Fiber Reconciler 每执行一段时间, 都会将控制权交回给浏览器, 这需要 优先级 来确定插队的规则
  synchronous, // 与之前的Stack Reconciler操作一样, 同步执行,  如 文本框的输入
  task,        // 本次调度结束需完成的任务, 在 nextTick 之前执行
  animation,   // 动画过渡, 下一帧之前执行
  high,        // 需要尽快执行, 如 交互反馈
  low,         // 稍微延迟执行也没关系, 如数据获取
  offscreen,   // 下一次 render 时或 scroll 时才执行

## 协调器
- fiber: 
  - 概述: 将 diff 拆成任务队列, 以解决 diff 计算量大阻塞渲染的问题
    - 背景: JS运算、页面布局、渲染 都是在浏览器主线程中进行的, 因为数据发生变化时, react 的更新是从根节点开始的, 在节点数量大的情况下 diff 耗时较长, 可能造成丢帧
    - 解决思路: 将 diff 分成多个部分, 在浏览器空闲时完成计算, 避免阻塞UI渲染
    - 实现方案: requestIdleCallback + 任务队列, 在 requestIdleCallback 的回调中判断是否有空余时间, 如有则从任务队列里取出任务并执行
      - requestIdleCallback(cd, { timeout: 0 }) 会在浏览器空闲时调用, 并返回给传入的回调一个对象, 内含 
        didTimeout: 是否超过 timeout 的boolean
        timeRemaining() 用于获取当前帧剩余空闲时间的 函数
        ```
          function workLoop(deadline) {
            // 这个 while 循环会在 任务执行完 或者 时间到了 的时候结束
            while(nextUnitOfWork && deadline.timeRemaining() > 1) {
              nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
            }
            // 如果任务还没完, 但是时间到了, 我们需要继续注册requestIdleCallback
            requestIdleCallback(workLoop);
          }

          // performUnitOfWork 用来执行任务, 参数是我们的当前 fiber 任务, 返回值是下一个任务
          function performUnitOfWork(fiber) {
            // 根节点的 dom 就是 container, 如果没有这个属性, 说明当前 fiber 不是根节点
            if(!fiber.dom) {
              fiber.dom = createDom(fiber);   // 创建一个DOM挂载上去
            } 

            // 如果有父节点, 将当前节点挂载到父节点上
            if(fiber.return) {
              fiber.return.dom.appendChild(fiber.dom);
            }

            // 将我们前面的vDom结构转换为fiber结构
            const elements = fiber.children;
            let prevSibling = null;
            if(elements && elements.length) {
              for(let i = 0; i < elements.length; i++) {
                const element = elements[i];
                const newFiber = {
                  type: element.type,
                  props: element.props,
                  return: fiber,
                  dom: null
                }

                // 父级的child指向第一个子元素
                if(i === 0) {
                  fiber.child = newFiber;
                } else {
                  // 每个子元素拥有指向下一个子元素的指针
                  prevSibling.sibling = newFiber;
                }

                prevSibling = newFiber;
              }
            }

            // 这个函数的返回值是下一个任务, 这其实是一个深度优先遍历
            // 先找子元素, 没有子元素了就找兄弟元素
            // 兄弟元素也没有了就返回父元素
            // 然后再找这个父元素的兄弟元素
            // 最后到根节点结束
            // 这个遍历的顺序其实就是从上到下, 从左到右
            if(fiber.child) {
              return fiber.child;
            }

            let nextFiber = fiber;
            while(nextFiber) {
              if(nextFiber.sibling) {
                return nextFiber.sibling;
              }

              nextFiber = nextFiber.return;
            }
          }
          requestIdleCallback(workLoop);
        ```
      - 任务队列: 
  
  - FiberNode 结构: 静态数据标识 + 指针信息 + 动态工作单元
    ```
      function FiberNode(
        tag: WorkTag,
        pendingProps: mixed,
        key: null | string,
        mode: TypeOfMode,
      ) {
        // 作为静态数据结构的属性, 描述组件信息
        this.tag = tag;                         // 组件类型: Function/Class/Host...
        this.key = key;
        this.elementType = null;                // 大部分情况同type，某些情况不同，比如 FunctionComponent 使用React.memo包裹
        this.type = null;                       // FunctionComponent => 函数本身, ClassComponent => Class, HostComponent => DOM节点tagName
        this.stateNode = null;                  // 真实DOM节点, 等同于 Vue 中的 $el

        // 用于连接其他 Fiber节点形成 Fiber树
        this.return = null;                     // 父节点
        this.child = null;                      // 首个子节点
        this.sibling = null;                    // 兄弟节点
        this.index = 0;

        this.ref = null;

        // 作为动态的工作单元的属性
        this.pendingProps = pendingProps;
        this.memoizedProps = null;
        this.updateQueue = null;
        this.memoizedState = null;
        this.dependencies = null;

        this.mode = mode;

        // 保存本次更新会造成的DOM操作
        this.effectTag = NoEffect;              
        this.nextEffect = null;

        this.firstEffect = null;
        this.lastEffect = null;

        // 调度优先级相关
        this.lanes = NoLanes;
        this.childLanes = NoLanes;

        // 指向该fiber在另一次更新时对应的fiber
        this.alternate = null;                  // 有点像 _vnode 和 $vnode
      }
    ```
    所有的组件、DOM节点都会抽象成一个 Fiber 节点, 基于 虚拟DOM 加上额外的信息生成, 本质上是 链表, 特点为
    1. 父节点 指向 第一个子节点
    2. 子节点 指向 下一个子节点
    3. 所有子节点 都指向 自己的父节点

  - Fiber Tree 构建过程: 
    1. 双缓存: 当协调完成, 交给渲染器后, 两棵树身份互换
    2. 可中断的递归: while 循环，从根节点开始，深度下探到叶子节点，再找兄弟节点，然后回到父节点
      ```
        function workLoop(deadline) {
          // 这个 while 循环会在 任务执行完 或者 时间到了 的时候结束
          while(nextUnitOfWork && deadline.timeRemaining() > 1) {
            nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
          }
          // 如果任务还没完, 但是时间到了, 我们需要继续注册requestIdleCallback
          requestIdleCallback(workLoop);
        }
      ```
    3. [Fiber 的工作内容](https://react.iamkasong.com/process/reconciler.html)
      递(beginWork): 传入当前 workInProgressFiber, 创建子 workInProgressFiber.child, 对于 update 流程，还会为 fiber 创建 effectTag
      归(completeWork): 处理 props、生成真实 DOM 树
      effectList 提取: 在构建 fiber 树的过程中，对存在 effectTag 标识的节点，会单独存放在 firstEffect 链表上，在后续 commit 给渲染器时，可以加快更新的效率
    4. Fiber Reconciler 执行的两个阶段:
      生成 Fiber 树, 得出需要更新的节点信息。这一步是一个渐进的过程, 可以被打断
      将需要更新的节点一次过批量更新, 这个过程不能被打断
  
  - diff: 浮标递增法
    优化点: 
      tree diff: 同层比较
      component diff: 拥有相同类的两个组件将会生成相似的树形结构, 如果 组件的类型不同 将跳过element diff, 直接卸载组件
      element diff: 对于同一层级的一组子节点, 它们可以通过唯一 key 进行区分
        移动: 
          依次取新列表中的节点, 当 新节点在旧列表中的下标index < 浮标lastIndex 时移动旧节点到lastIndex并patch
          每次比较 lastIndex = max(lastIndex, index)
        新增: 新节点在旧列表中找不到, 则在 之前插入新建的节点
        删除: 新列表遍历完, 再遍历旧列表, 删除多余的节点

## 渲染器
- React.createElement: 将 JSX (<div></div>) 通过 babel 转换成 待执行的 React.createElement({ 'div' }), 用于生成 虚拟DOM (ReactElement)
- ReactElement: 虚拟DOM对象, 类似于 vnode, 本质上是一个js对象, 用于描述一个真实dom, 关键属性有 type、props、children 等
- ReactDOM.render: 目的是将 虚拟DOM 对象转换成 真实DOM
  ```
    function render(vDom, container) {
      let dom;
      // 检查当前节点是文本还是对象
      if(typeof vDom !== 'object') {
        dom = document.createTextNode(vDom)
      } else {
        dom = document.createElement(vDom.type);
      }

      // 将 vDom 上除了 children 外的属性都挂载到真正的 DOM 上去
      if(vDom.props) {
        Object.keys(vDom.props)
          .filter(key => key != 'children')
          .forEach(item => {
            dom[item] = vDom.props[item];
          })
      }
      
      // 如果还有子元素, 递归调用
      if(vDom.props && vDom.props.children && vDom.props.children.length) {
        vDom.props.children.forEach(child => render(child, dom));
      }

      container.appendChild(dom);
    }
  ```

## setState 做了什么
  1. setState 的作用是将 传入的对象 与 当前状态 合并, 重新进行渲染
  2. 
  在 setState 调用时, 会创建组件对应的 fiber 节点, 生成 update对象 进入 updateQueue
    update 对象结构: {
      eventTime,          // 更新创建的时间
      lane,               // 更新优先级
      suspenseConfig,     // 任务挂起相关
      tag: UpdateState,   // 更新是哪种类型（UpdateState, ReplaceState, ForceUpdate, CaptureUpdate）
      payload: null,      // 更新所携带的状态
      callback: null,
      next: null,         // 指向下一个update的指针
    }

## 组件生命周期
  组件有三种状态: 挂载中、更新中、卸载中；分别对应 mountComponent、updateComponent、unmountComponent, 除卸载外, 各自提供了两种处理方法 will、did
  - 挂载
    * 组件的 constructor
    ？ getDerivedStateFromProps
    * render
      只返回需要渲染的东西
    * componentDidMount
      组件挂载完成时调用, 此时可以拿到 DOM
      官方建议将异步请求放在这个阶段

  - 更新
    ？getDerivedStateFromProps
    * shouldComponentUpdate (nextProps, nextState) => boolean(是否触发重新渲染)
    * render
    ？getSnapshotBeforeUpdate (prevProps, prevState) 
    => componentDidUpdate (prevProps, prevState, snapshot)

  - 卸载
    componentWillUnmount: 清除定时器, 取消网络请求, 清理无效的DOM元素等垃圾清理工作

## React 逻辑复用方式
- Mixin
  本质上是 merge 的过程
  问题是多个 mixin 容易产生依赖和冲突, 不易维护

- HOC (高阶函数)
  本质上是纯函数, 在不修改传入内容的前提下完成扩展, 如 props 扩展：<WrappedComponent {...this.props} {...newProps}/>
  问题是多层嵌套时调试会比较困难, 比较依赖约定, 比如透传不相关的 props

- Hook
  为函数组件提供维护状态的能力, 扁平式地处理逻辑复用, 提高代码的可读性

- 参考资料
  [从Mixin到HOC再到Hook](https://mp.weixin.qq.com/s?__biz=Mzk0MDMwMzQyOA==&mid=2247490057&idx=1&sn=e7a9abb4df2fb7f7baf406dbb20d8313&source=41#wechat_redirect)

## 题库
- setState 是同步还是异步？ https://zhuanlan.zhihu.com/p/39512941
  在 合成事件 和 钩子函数 中, setState 是异步的, 因为 React 为了减少 无意义的 render, 用更新队列进行了缓存
  在 原生事件 和 setTimeout 中是同步的, 因为上下文的丢失, 他们不参与合并

- 为什么有时连续多次 setState 只有一次生效？
  setState的时候react内部会创建一个updateQueue
  在最终的 performWork 中, 相同的 key 会被覆盖, 只会对最后一次的 setState 进行更新

- react 类组件 和 函数组件 的差异
  类组件面向对象, 需要更重的心智；函数组件面向过程, 注重逻辑的纯粹；
  函数组件没有 this、没有生命周期、没有 state, 只根据传入数据完成渲染, 功能更纯粹
  在 类组件 中, 同一职责的代码可能会被拆分到不同的生命周期函数中, 不同的生命周期也可能多次处理重复逻辑, 可维护性差
  在组件需要 包含内部状态 或者 使用到生命周期函数 的时候使用 类组件, 否则使用函数式组件

- React 如何实现自己的事件机制 ？
  为了解决 兼容性 和 DOM事件滥用 的问题, react自己实现了 合成事件
  组件挂载和更新阶段, 在事件中心注册了事件, 标识符是 事件类型(type)和组件标识(_rootNodeID)
  react没有将事件绑定在DOM上, 采用了事件委托的形式统一在 document 上派发事件
  原生事件是绑定在 dom 上的, 执行顺序相对 合成事件 靠前

- 为何 React事件 要自己绑定 this ？
  JSX在转换后, this 的指向发生了隐式丢失

- 为什么不要在循环、条件语句或嵌套函数中调用 Hooks
  以 useState 为例, 其以 数组 管理render所用到的 state, 如果在条件语句中使用, 则有可能导致 前后数组长度不一致 而更新错乱



## 备用参考资料
[fiber 的实现原理](https://www.cnblogs.com/dennisj/p/13183332.html)
[fiber 架构原理](https://zhuanlan.zhihu.com/p/525244896)
