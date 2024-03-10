## Fiber 的作用
  JS线程与渲染线程是互斥的
  避免 过于庞大的组件计算 阻塞 浏览器渲染 而导致丢帧

## Fiber 的主要工作
  1. 为任务划分优先级
  2. 将组件渲染拆分成多个更新任务，以链表形式组织，以方便中断和恢复
  3. 

## 两颗 Fiber 树
  

## 
https://react.iamkasong.com/preparation/idea.html#react%E7%90%86%E5%BF%B5

## FiberNode 结构
  - 树结构相关
    child: 子节点
    sibling: 下一个兄弟节点
    return: 父节点

  - 描述信息
    tag: 
    type: DOM/组件 类型
    key: 唯一标识符
    

  ```
  {
    this.tag = tag;
    this.elementType = null;
    this.type = null;
    this.stateNode = null; // Fiber

    this.index = 0;
    this.ref = null;
    this.pendingProps = pendingProps;
    this.memoizedProps = null;
    this.updateQueue = null;
    this.memoizedState = null;
    this.dependencies = null;
    this.mode = mode; // Effects

    this.effectTag = NoEffect;
    this.nextEffect = null;
    this.firstEffect = null;
    this.lastEffect = null;
    this.expirationTime = NoWork;
    this.childExpirationTime = NoWork;
    this.alternate = null;
  }
  ```

## 简易的 fiber 流程
  - 

  ```
    function workLoopConcurrent() {
      // 持续执行更新任务，直到 调度器 需要我们让步
      while (workInProgress !== null && !shouldYield()) {
        workInProgress = performUnitOfWork(workInProgress);
      }
    }

    performUnitOfWork(unitOfWork: Fiber) {

    }

    shouldYield() {}


  ```