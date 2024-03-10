## react 触发 render 的三种方式
  1. ReactDOM.render 
  2. setState
  3. forceUpdate

## ReactDOM.render 的三个阶段
 1. 初始化阶段: 完成 Fiber树 中基本实体的创建
    - legacyRenderSubtreeIntoContainer: 创建 root + fiberRoot
      container: 容器，一般是 <div id="app" /> 对应的 DOM对象
      root: FiberRootNode 节点实例
      fiberRoot: Fiber 树的头部根结点
        containerInfo: 对应 container，即 DOM容器
        current: 对应一个 FiberNode 节点实例

    - updateContainer
      - 获取当前时间
      - 计算过期时间: 根据 fiber.mode 
      - createUpdate: 创建一个更新任务
        update 是链表结构，其 next属性 update 自身 // todo ???
        确定 update任务 优先级 (立即、用户阻塞级别、常规级别、低优先级、空闲级别)
      - update 进入队列
    
    - scheduleUpdateOnFiber: 调度器开始调度
      - markUpdateTimeFromFiberToRoot: 找到rootFiber 并 遍历更新子节点的expirationTime
        // todo
      - 

      确定当前 Fiber 节点的优先级
      结合 lane（优先级），创建当前 Fiber 节点的 update 对象，并将其入队
      调度当前节点（rootFiber）

 2. render 阶段: 完成 Fiber树 的构建
   - performSyncWorkOnRoot: render阶段开始

   - beginWork: 负责创建 Fiber 节点

   - completeWork: 负责将 Fiber 节点映射为 DOM节点

   - finishSyncRender: render阶段结束

 3. commit 阶段



