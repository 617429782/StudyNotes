## 盒模型
  1. 标准盒模型 content-box 的 width 指的是 content 部分
  2. IE盒模型 border-box 的 width 指的是 content + padding + border 部分
  3. 指定盒模型: box-sizing: border-box / content-box

## CSS选择器
  !important > 内联样式 > ID选择器 > 类选择器/属性选择器/伪类 > 标签选择器 > 通配符选择器(*)/选择符(+、>、~等)/逻辑组合伪类(:not、:is、:where)
  选择器等级相同时, 总值大的优先, 总值一样后定义的优先, 选择器不能跨等级

## 层叠上下文
  1. 层叠上下文特性: 谁大谁上, 后来居上, 子跟随父
  2. 层叠上下文的顺序
  3. 产生层叠上下文的方式

## BFC 块级上下文 
  1. BFC特性
    1. BFC 内元素不会超出包含块
    2. 块级元素 会在 上下文内部 垂直排列
    3. 同一个BFC内部的 相邻块级元素 会产生 margin重叠
    4. 每个块级元素的 margin-box 左边与 包含块的 border-box 左边相接触, 除非该块级元素生成了新的BFC
    5. 计算BFC高度时, 会考虑浮动元素

  2. 触发BFC的规则 
    1. 根元素 html
    2. float
    3. position: absolute/fixed
    4. overflow 不为 visible
    5. display
      inline-block
      flex / inline-flex
      grid / inline-grid
    6. (其他一些冷门属性) https://cloud.tencent.com/developer/article/1687921
      display: table / inline-table / table-cell / table-caption / table-row / table-row-group / table-header-group / table-footer-group
      column-count 或者 column-width 不为 auto
  
  3. BFC块级上下文的应用
    1. 解决浮动的高度塌陷(利用BFC包含float高度)
    2. 防止margin重叠(产生新的BFC即可)
    3. 列布局(利用BFC不与float重叠, 即不产生环绕)

## 浮动
  1. 浮动特性
    1. 浮动元素脱离正常文档流, 在遇到 包含他的边框 或 其他浮动元素的边框 时停止
    2. 浮动元素的高度不计入父元素, 因此会产生 "高度塌陷"
    3. 浮动可以产生文字环绕效果, 这是难以模拟的

  2. 清除浮动
    1. 使用BFC包裹浮动元素
    2. 通过clear: both, 清除当前元素 前面元素 浮动产生的影响

## 固定+自适应多列布局
  float+width / overflow:hidden (产生BFC)
  float+width / margin-left: width ( float 会与 非BFC 重叠产生环绕)
  width / absolute + left: width
  width / flex: 1 (利用 flex-grow 的放大比例, 平分剩余空间)

## 水平/垂直 居中
  固定宽高 + absolute + 四边取0 + margin: auto
  justify-content: center + align-items: center
  absolute 50% + trasform: translateX/Y(-50%)

## flex布局 (容器+子项) https://www.ruanyifeng.com/blog/2015/07/flex-grammar.html    
  - 容器: 设为 Flex布局以后, 子元素的 float、clear、vertical-align 属性失效
    1. flex-flow: direction/wrap 的简写, 默认 (row nowrap)
      flex-direction: 子项排列方式 row(水平) / row-reverse(水倒) / column(垂直) / column-reverse(垂倒)
      flex-wrap: 处理换行 nowrap(不换行) / wrap(倒左金字塔) / wrap-reverse(正左金字塔);
    
    2. justify-content: 子项在 主轴 上的对齐方式(默认横向)
      flex-start(默认): 左对齐
      flex-end: 右对齐
      center: 居中
      space-between: 两端对齐(贴边), 子项间隔相等
      space-around: 两端对齐(留白), 子项间隔 是 留白 2倍

    3. align-items: 子项在 交叉轴 上的对齐方式(默认纵向)
      flex-start: 顶对齐
      flex-end: 底对齐
      center: 居中对齐
      baseline: 基线对齐
      stretch(默认): 子项不设高度时, 跟随容器

    4. align-content: 多行在垂直方向的排列 (参考justify-content)
      flex-start
      flex-end 
      center
      space-between
      space-around
      stretch
      
  - 子项
    1. order: 排列顺序, 小的靠前
    2. flex: (grow | shrink | basis)
      flex-grow: 放大比例(基于剩余空间, 0不放大)
      flex-shrink: 缩小比例(空间不足的情况, 0不缩小)
      flex-basis: 预期大小(默认auto即原始大小, 可参考width, 优先级比width高)
    3. align-self: align-items 的单独处理

  - 常用案例
    水平垂直居中: 对容器 justify-content: center + align-items: center
    定宽+自适应布局: 定宽子项 width + 自适应子项 flex: 1

## grid http://www.ruanyifeng.com/blog/2019/03/grid-layout-tutorial.html      
  

## 移动端 1px / 0.5px
  1. 单边 不支持圆角
    1. 伪类缩放 transform: scaleX/Y(0.5)
    2. 渐变背景 background-image: linear-gradient(0deg, #f00 50%, transparent 50%);
  2. 四边 支持圆角
    1. 伪类缩放 伪类宽高 设为 元素宽高 的 2倍, 设置 1px的border 后 scale(0.5)
    2. 阴影 box-shadow: 0 0 0 0.5px red;

## Sass / Less

## 媒体查询
  // todo 用得不多, 一般用于适配不同大小的屏幕, 在特定的宽度区间使用指定样式



  