- TypeScript
  - 类型注解：为 函数参数 或者 变量 添加类型约束 用法 let x: number = 1
    基础类型 string / boolean / number
    引用类型 object
    数组：相同类型的不定长元素集合
      number[] / Array<number>
    元祖：数量和类型已知的数组，各元素的类型不必相同，访问越界的元素将会使用 联合类型 替代
      let arr: [string, number] = ['1', 2]
    枚举：一组带名字的不重复常量
      enum Color {Red, Green, Blue}; Color.Red = 5    // error

      数字枚举：自增长特性
        enum Color {Red, Green, Blue}       // 分别是 0，1，2
        enum Color {Red = 1, Green, Blue}   // 分别是 1，2，3
        enum Color {Red, Green = 2, Blue}   // 分别是 0，2，3
      字符串枚举：必须初始化

      反向映射：通过值取键
        enum Color {Red = 1, Green, Blue}
        Color.Red     // 1
        Color[1]      // Red

    any 
    void / null / undefined / never

  - 类型断言：类似于强转
    断言支持两种语法: as语法 和 尖括号语法
    ```
      let someValue: any = "this is a string";
      let strLength: number = (<string>someValue).length; // 尖括号语法
      let strLength: number = (someValue as string).length; // jsx 仅支持 as 语法
    ```

  - 类型推论：为没有明确指出类型的地方提供类型
    产生时机：变量初始化、默认参数值、函数返回值

  - 类型兼容性：赋值的 目标对象(右)属性 要覆盖 源对象(左)属性，这个比较是递归的，检查所有成员属性及其子属性
    变量赋值、函数传参：比较属性名
      interface Named { name: string; }; 
      let x: Named;
      let y = { name: 'Alice', location: 'Seattle' };
      x = y;                                              // 不报错，因为 x 的所有属性都在 y 的属性范围内
      function greet(n: Named) {};  greet(y);             // 不报错，同上
      let z = { name2: 'Alice', location: 'Seattle' };
      x = z;                                              // 报错，x 的属性未被 z 覆盖
      function greet(n: Named) {};  greet(z);             // 报错，同上
    函数类型的赋值：比较 参数类型 和 返回值类型
      参数不同，返回值相同时，比较 参数类型
        let x = (a: number) => 0;
        let y = (b: number, s: string) => 0;
        y = x;  // OK，x 的所有参数类型在 y 的对应位置都有体现 
        x = y;  // Error，y 的 s 参数未在 x 中定义
      参数相同，返回值不同时，比较返回值类型，参考变量赋值
        let x = () => ({name: 'Alice'});
        let y = () => ({name: 'Alice', location: 'Seattle'});
        x = y; // OK
        y = x; // Error, because x() lacks a location property

  - 接口：类似于结构体的概念, 注意各属性分割使用 ; 而非对象的 , 
    interface Person {
      name: string;
      readonly sex: string;  // 只读属性
      brother?: string[];    // 可选属性
    }
    
    函数类型接口，可视为一个只有 参数列表 和 返回值类型 的函数定义
      函数类型接口不要求参数名匹配，但要求对应位置的参数类型匹配
      interface SearchFunc {
        (source: string, subString: string): boolean;
      }
      let mySearch: SearchFunc;
      mySearch = function(src: string, sub: string): boolean {  // 参数名可以于定义不同，参数类型与返回值要相同
        let result = src.search(sub);
        return result > -1;
      }
    
    可索引的类型

    接口继承

  类
    继承 extends，类A继承了类B后，A的实例对象a就拥有了B中定义的属性和方法

    可以在class的constructor构造函数中使用public描述的参数，等同于创建了同名变量

    可以实现接口，使得类符合一定的规范
      interface ClockInterface {
        currentTime: Date;
      }

      class Clock implements ClockInterface {
        currentTime: Date;
        constructor(h: number, m: number) { }
      }



  命名空间

  泛型：记录类型的变量
    范型函数 function fn<T>(arg) {} 接受 类型参数 T 和 参数 arg
      function fn<T>(arg: T): T {     // T 记录了函数的入参类型
        return arg;
      }
      let output = fn<string>("123"); // "123" 函数传入的是string类型，因此return的也是string类型
      let output = fn("123");         // "123" 因为 类型推论 的存在，编译器会判断入参的类型，因此可以省略 <>

  高级类型
    交叉类型：多个类型合并为一个类型，

    联合类型



 