## react 的三层架构
- 虚拟DOM: 负责用JS描述DOM对象
- 调度层: 负责调用组件生命周期、对虚拟DOM进行diff
- 渲染层: 将虚拟DOM渲染成DOM对象

https://www.cnblogs.com/dennisj/p/13183332.html

## react 流程
- React.createElement: 将 JSX (<div></div>) 通过babel转换成 待执行的 React.createElement({ 'div' }), 用于生成 虚拟DOM (ReactElement)
  ReactElement: 虚拟DOM对象, 类似于 vnode, 本质上是一个js对象, 用于描述一个真实dom, 关键属性有 type、props、children 等
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

      // 将vDom上除了children外的属性都挂载到真正的DOM上去
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

## diff: 浮标递增法
  优化点: 
    tree diff: 同层比较
    component diff: 拥有相同类的两个组件将会生成相似的树形结构, 如果 组件的类型不同 将跳过element diff, 直接卸载组件
    element diff: 对于同一层级的一组子节点, 它们可以通过唯一 key 进行区分
      移动: 
        依次取新列表中的节点, 当 新节点在旧列表中的下标index < 浮标lastIndex 时移动旧节点到lastIndex并patch
        每次比较 lastIndex = max(lastIndex, index)
      新增: 新节点在旧列表中找不到, 则在 之前插入新建的节点
      删除: 新列表遍历完, 再遍历旧列表, 删除多余的节点

## fiber: 将 diff 拆成任务队列, 以解决 diff 计算量大阻塞渲染的问题
- 背景: JS运算、页面布局、渲染 都是在浏览器主线程中进行的, 当调用 setState 时, 在节点数量大的情况下 diff 耗时较长, 可能造成丢帧
- 解决思路: 将 diff 分成多个部分, 在浏览器空闲时完成计算, 避免阻塞UI渲染
- 实现方案: requestIdleCallback + 任务队列, 在 requestIdleCallback 的回调中判断是否有空余时间, 如有则从任务队列里取出任务并执行
  - requestIdleCallback(cd, { timeout: 0 }) 会在浏览器空闲时调用, 并返回给传入的回调一个对象, 内含 
    didTimeout: 是否超过 timeout 的boolean
    timeRemaining() 用于获取当前帧剩余空闲时间的 函数

    ```
      function workLoop(deadline) {
        // 这个while循环会在任务执行完或者时间到了的时候结束
        while(nextUnitOfWork && deadline.timeRemaining() > 1) {
          nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
        }
        // 如果任务还没完, 但是时间到了, 我们需要继续注册requestIdleCallback
        requestIdleCallback(workLoop);
      }

      // performUnitOfWork用来执行任务, 参数是我们的当前fiber任务, 返回值是下一个任务
      function performUnitOfWork(fiber) {
        // 根节点的dom就是container, 如果没有这个属性, 说明当前fiber不是根节点
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


- 任务优先级: Fiber Reconciler 每执行一段时间, 都会将控制权交回给浏览器, 这需要 优先级 来确定插队的规则
  synchronous, // 与之前的Stack Reconciler操作一样, 同步执行,  如 文本框的输入
  task,        // 本次调度结束需完成的任务, 在next tick之前执行
  animation,   // 动画过渡, 下一帧之前执行
  high,        // 需要尽快执行, 如 交互反馈
  low,         // 稍微延迟执行也没关系, 如数据获取
  offscreen,   // 下一次 render 时或 scroll 时才执行
- Fiber Reconciler 执行的两个阶段:
  生成 Fiber 树, 得出需要更新的节点信息。这一步是一个渐进的过程, 可以被打断
  将需要更新的节点一次过批量更新, 这个过程不能被打断
- Fiber 树: 
  所有的组件、DOM节点都会抽象成一个 Fiber 节点, 基于 虚拟DOM 加上额外的信息生成, 本质上是 链表, 特点为
  父节点 指向 第一个子节点
  子节点 指向 下一个子节点
  所有子节点 都指向 父节点
  这样不管在哪里中断, 都可以通过中断的那个任务, 找到下一个需要执行的任务 子 -> 兄弟 -> 父节点

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
  组件有三种状态: 挂载中、更新中、卸载中
  分别对应 mountComponent、updateComponent、unmountComponent, 除卸载外, 各自提供了两种处理方法 will、did

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

## React 逻辑复用方式 https://mp.weixin.qq.com/s?__biz=Mzg2NDAzMjE5NQ==&mid=2247484193&idx=1&sn=0c152afb3566f2b600b0e218d5d24017&chksm=ce6ec78df9194e9b3e0e4f18706fd84be22f0f84d1e7554892e169a30265c6fa154731c1c7a3&scene=178&cur_album_id=1403155327595495424#rd
- Mixin
  本质上是将多个对象合并成一个对象

- HOC (高阶函数)
  属性代理: 包裹原组件, 在不修改原组件的情况下扩展其 prop <WrappedComponent {...this.props} {...newProps}/>
  反向继承: 通过继承, super的特性完成函数劫持

- Hook

## 题库
- setState 是同步还是异步？ https://zhuanlan.zhihu.com/p/39512941
  在 合成事件 和 钩子函数 中, setState 是异步的, 因为React为了减少 无意义的render，用更新队列进行了缓存
  在 原生事件 和 setTimeout 中是同步的, 因为上下文的丢失，他们不参与合并

- 为什么有时连续多次 setState 只有一次生效？
  setState的时候react内部会创建一个updateQueue
  在最终的performWork中, 相同的key会被覆盖, 只会对最后一次的setState进行更新

- react 类组件 和 函数组件 的差异
  函数组件没有 this、没有生命周期、没有 state, 只根据传入数据完成渲染, 功能更纯粹
  在 类组件 中, 同一职责的代码可能会被拆分到不同的生命周期函数中, 不同的生命周期也可能多次处理重复逻辑, 可维护性差
  在组件需要 包含内部状态 或者 使用到生命周期函数 的时候使用 类组件, 否则使用函数式组件

- React 如何实现自己的事件机制 ？
  为了解决 兼容性 和 DOM事件滥用 的问题, react自己实现了 合成事件
  组件挂载和更新阶段, 在事件中心注册了事件, 标识符是 事件类型(type)和组件标识(_rootNodeID)
  react没有将事件绑定在DOM上, 采用了事件委托的形式统一在 document 上派发事件
  原生事件是绑定在 dom 上的, 执行顺序相对 合成事件 靠前

- 为何 React事件 要自己绑定 this ？
  JSX在转换后, this的指向发生了隐式丢失

- 为什么不要在循环、条件语句或嵌套函数中调用 Hooks
  以 useState 为例，其以 数组 管理render所用到的 state，如果在条件语句中使用，则有可能导致 前后数组长度不一致 而更新错乱