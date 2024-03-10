## 词法分析
  一般有两种写法: 正则(vue), 状态机

## 简易 html 解析引擎
  ```
    输入 `<div id='abc'><span>123</span>456</div>`
    输出 
      {
        name: 'div',
        attrs: {
          id: 'abc'
        },
        children: [
          {
            name: 'span',
            attrs: {},
            children: [
              {
                type: 'text',
                text: '123'
              }
            ]
          }, {
            type: 'text',
            text: '456'
          }
        ]
      }
  ```

  实现思路
  ```
    维护一个栈 stack
    逐字读取 html 片段
    读取到 <div id='abc'>, node 入栈: [div]
    读取到 <span>, node 入栈, 栈顶节点增加子节点: [div->[span], span]
    读取到 文本节点 123, 栈顶节点增加子节点 [div->[span->[123]], span->[123]]
    读取到 </span>, node 出栈 [div->[span->[123]]
    读取到 文本节点 456, 栈顶节点增加子节点 [div->[span->[123], 456]]
    读取到 </div>, node 出栈, 得到最终节点数 div->[span->[123], 456]

    标签头闭合时 node 入栈, 同时建立父子关系
    标签尾闭合时 node 出栈,
    栈中元素是父子关系, 这是一个深度优先的构建顺序

  ```

  实现代码 (正则)
  ```
    const startReg = /^<(?!\/)(\n|.)*?>/
    const endReg = /^<\/(\n|.)*?>/

    let html = `<div id='abc'><span>123</span>456</div>`;
    const stack = [];
    let root = null;
    while (html) {
      const unitStart = html.indexOf('<');
      if (unitStart === 0) { // 标签节点
        if (html.match(startReg)?.[0]) {
          const startTag = html.match(startReg)?.[0];
          stack.push(parseNode('start', startTag))
          html = html.substring(startTag.length)
          continue;
        }
        if (html.match(endReg)?.[0]) {
          const endTag = html.match(endReg)?.[0];
          html = html.substring(endTag.length)
          const node = stack.pop();
          if (stack.length) {
            stack.slice(-1)[0].children.push(node)
          } else {
            root = node
          }
          continue;
        }
      } else {  // 文本节点
        const node = parseNode('text', html.substring(0, unitStart));
        stack.length && stack.slice(-1)[0].children.push(node)
        html = html.substring(unitStart)
        continue;
      }
    }

    function parseNode(type, html) {
      if (type === 'text') {
        return {
          type: 'text',
          text: html
        }
      } else if (type === 'start') {
        const arr = html.slice(0, -1).split(' '); // 去掉末尾的 >
        const result = {
          name: arr[0].substring(1),
          attrs: {},
          children: []
        }
        arr.slice(1).forEach(v => {
          const [key, val] = v.split('=');
          result.attrs[key] = val.match(/(?<=\'|\")(\n|.)*?(?=\'|\")/)?.[0];  // 取 '' 或者 "" 间的属性值
        })
        return result;
      }
    }
  ```
