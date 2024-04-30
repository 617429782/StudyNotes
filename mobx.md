## 定位：响应式状态管理库 https://cn.mobx.js.org/

## API
- (@)computed 
- autorun(fn, options) => 类比 immediate 为 true 的 watch  
  - delay: 防抖
  - scheduler
- reaction(() => data, (data, reaction) => { sideEffect }, options?) => autorun 的变种，以函数来控制传入参数是否执行，而不是待执行函数的依赖
  - fireImmediately: 立即执行
  - delay: 防抖
  - equals
  - scheduler
- when(predicate, effect) => 等待 predicate 为 true 时，执行 effect
