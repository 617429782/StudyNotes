## hooks 简介
  - 定义
    Hook 是一些可以让你在 函数组件 里钩入 状态 及 生命周期等特性 的函数
    前提: 函数组件 内使用
    本质: 函数
    作用: 扩展组件的 state 与 生命周期

    复用代码更简单，职责单一
    组合 的方式更优雅，与 render、props、高阶组件HOC 不同，hooks不引入嵌套，也没有 mixins 的负面影响
  
  - 适用场景
    hook 只能在函数最外层调用，不可在循环、条件判断、子函数 中调用 (原因见题库)
    hook 只能在 函数组件 与 自定义hook 中使用

## 常用的 hooks https://zh-hans.reactjs.org/docs/hooks-reference.html
  - useState
    - 基础用法: 返回一对值，当前状态 与 更新状态的函数
      ```
        const [count, setCount] = useState(0)
        setCount(1)
      ```
    - 实现原理: 闭包一个 数组 管理 本次render 的 所有state
      ```
        const React = (function () {
          const _hooks = []  // 使用数组, 每次调用 useState 都移动索引供下一次调用，每个索引对应组件的 state
          let _index = 0     // hooks 的索引

          return {
            render(Component) {
              const c = Component();
              c.render();
              _index = 0; // render 之后重置
              return c
            },
            useState(initVal) {
              // 首次render用初始值，后续render调用useState时，将沿用上一次render的状态
              const state = _hooks[_index] === undefined ? initVal : _hooks[_index]
              const index = _index  // 闭包使得组件中的 setState 可以通过索引找到 _hooks 中存放的值
              function setState(val) {
                _hooks[index] = val
              }
              _index++
              return [state, setState]
            }
          }
        })()

        // 在组件中使用
        const { useState, render } = React;
        function Counter() {
          const [count, setCount] = useState(0)

          return {
            render() {
              console.log(count)
            },
            add() {
              // setTimeout(() => {console.log('5秒后', count)}, 5000)
              setCount(count + 1)
            }
          }
        }
        const c = render(Counter) // 0
        c.add()
        render(Counter) // 1
      ```
    - 回调模式: 
      setCount 支持回调模式，这可以移除部分场景下对 count 变量的依赖
      ```
        setCount(c => c + 1);
      ```

  - useReducer: 当需要使用多个 state 来实现一个功能时，不妨使用 useReducer，将 state 整合成一个对象
    - 基础用法 (state, action) => newState
      用法与 useState 接近，通过dispatch触发更新，接收当前 state 和 触发动作action，计算并返回新的 state
      新旧 state 通过 Object.js 比较，相同时不会触发render，因此不能直接修改旧state的属性，需要返回一个新的state
      ```
        // 参数列表分别对应 更新策略、初始值、返回初始值的函数
        // const [state, dispatch] = useReducer(reducer, initialArg, init);
        const [state, dispatch] = useReducer((state, action) => {
          switch (action.type) {
            case 'increment':
              return { count: state.count + 1 };
            case 'decrement':
              return { count: state.count - 1 };
            default:
              throw new Error();
          }
        }, { count: 0 });

        dispatch({type: 'decrement'})
      ```

    - 实现原理
 
  - useEffect / useLayoutEffect: 在 render 的 之后/之前 执行
    - 基础用法: 回调 + 依赖数组
      - 回调: 可以return一个 函数，其会在组件销毁时触发
        ```
        useEffect(() => {
          document.title = `You clicked ${count} times`;
          // 在组件销毁时触发，对于副作用操作，如 Event.on/off、clearInterval 非常有用
          return () => { document.title = `You clicked times` }
        });
        ```
      - 依赖数组 ( null / 0 / n )
        不传: 每次render之后都触发回调
        长度为0的数组: 仅第一次render触发回调
        长度为n的数组: 数组里的每一项改变都会触发回调，浅比较
        ```
        useEffect(() => {}, [...deps])
        ```
    - 实现原理 // todo: 需要更细节的实现
      ```
        var deps;
        function useEffect(cb, _deps) {
          if (!_deps || !_deps.every((v, i) => deps[i] === v)) {
            cb();
            deps = _deps
          }
        }
      ```
    - capture value 特性: 每次render使用的props和state是独立且固定的
      ```
        useEffect(() => {
          const id = setInterval(() => {
            document.title = `You clicked ${count} times`;
            // 因为 capture value 特性，定时器每次执行拿到的 count 都是首次render的 0，相当于 定时器只为那一次render的状态快照服务
            // 虽然 定时器在不断执行，但因为之后 setCount 的值都一样，因此没有触发 render
            setCount(count + 1);
          }, 1000);
          return () => clearInterval(id);
        }, []);
      ```

    - useLayoutEffect 用法与 useEffect 相同，区别在于 useEffec t执行时机在浏览器绘制之后，而 useLayoutEffect 是之前

  - useCallback / useMemo: 缓存 函数/返回值
    - 基础用法
      ```
        function Parent() {
          const [query, setQuery] = useState("react");

          // useCallback 将在 query 变化时，重新生成 fetchData 函数, 来触发 Child 中对 fetchData 的依赖
          const fetchData = useCallback(() => {
            const url = "https://hn.algolia.com/api/v1/search?query=" + query;
            // ... Fetch data and return it ...
          }, [query]);

          return <Child fetchData={fetchData} />;
        }

        function Child({ fetchData }) {
          let [data, setData] = useState(null);

          useEffect(() => {
            fetchData().then(setData);
          }, [fetchData]);
        }
      ```

    - 实现原理

  - useContext: 解决跨层级组件的状态共享
    - 基本用法 创建 + 包裹 + 消费
      当 Provider 提供的值变化时，所有 消费者组件 都会重新 render
      ```
        // 创建
        const ThemeContext = React.createContext('light');

        // 包裹
        class App extends React.Component {
          render() {
            // 第二步：使用 Provider 提供 ThemeContext 的值，Provider所包含的子树都可以直接访问ThemeContext的值
            return (
              <ThemeContext.Provider value="dark">
                <Toolbar />
              </ThemeContext.Provider>
            );
          }
        }
        
        function Toolbar(props) { // Toolbar 组件并不需要透传 ThemeContext
          return (
            <div>
              <ThemedButton />
            </div>
          );
        }

        // 使用
        function ThemedButton(props) {
          // 第三步：使用共享 Context
          const theme = useContext(ThemeContext);
          render() {
            return <Button theme={theme} />;
          }
        }
      ```

  - useRef: 提供DOM操作
    - 基础用法

    - 实现原理

  - useImperativeHandle
    

## 题库
  - 为什么 hooks 不可在循环、条件判断、子函数 中调用
    从 useState 的实现看出，react闭包了一个数组来管理 本次render 的所有 state，通过 index 把 state 和 数组中的值对应起来
    如果在条件语句中使用，就可能导致 本次render所用的state 在闭包数组中的位置 与上一次render使用的 对应不上，导致更新错乱
    (简单的说，一次render使用的state数量应该是固定的，如果在条件语句中使用，state的数量就不可控了)

  -  



