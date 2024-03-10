## 基本用法
- 基本类型
- 高级类型
- 泛型

## 装饰器 https://fynn90.github.io/2021/01/15/TS%E8%A3%85%E9%A5%B0%E5%99%A8/
  - 用法
    1. 在 tsconfig.json 的 compilerOptions 中 配置 experimentalDecorators: true，并且 target 版本不能小于 ES5
    2. @expression 必须返回 一个函数，其在编译时执行

  - 分类
    
## 类
- 成员可见性
  public: 默认共有，可在 class 的 内部 和 外部 被访问
  private: 私有成员，只允许在当前类的实例中被访问，子类不行
  protected: 受保护成员，允许在当前类和子类的实例中被访问
  static: 静态成员，可通过 类名.成员名 访问

## 题库
1. 装饰器的作用的实现原理

2. 私有、公有、保护属性的差异和实现原理

3. 