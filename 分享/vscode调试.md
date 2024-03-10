## 官方文档
  https://code.visualstudio.com/docs/editor/debugging#_launchjson-attributes

## 附加到进程
  {
    "version": "0.2.0",
    "configurations": [
      {
        "type": "node",
        "request": "attach",
        "name": "启动程序",
        "skipFiles": [
          "<node_internals>/**"
        ],
        "processId": "${command:PickProcess}"
      }
    ]
  }

## 供应商后台 webpack 热更新调试
  // 其他项目可以参考, 注意 cwd、program、args 根据 package.json 和 项目路径 配置
  {
    "version": "0.2.0",
    "configurations": [
      {
        "type": "node",
        "request": "launch",
        "name": "dev",
        "cwd": "${workspaceFolder}/packages/client/",
        "program": "${workspaceFolder}/node_modules/.bin/webpack-dev-server",
        "args": [
          "--mode",
          "development",
          "--config",
          "webpack.config.js",
        ],
        "env": {
          "NODE_ENV":"development",
        },
        "autoAttachChildProcesses": true,
        "stopOnEntry": true
      }
    ]
  }

## 调试 ast
  // 注意 cwd、args 根据 package.json 和 项目结构 配置
  - 分析过程
    1. 找到执行命令
      查看 package.json 的启动命令 start:dev, 其中调用了 ast dev
      因为 ast 是装在全局下的, 因此项目的 node_modules/.bin 中没有 ast 命令, 不能直接调用
      MAC下全局的 node_modules 目录地址: /usr/local/lib/node_modules
      (可以添加环境变量来调用全局/bin下的命令 export PATH=$PATH:/usr/local/bin)

    2. 找到执行文件
      添加环境变量后可以执行 ast dev, 但会提示找不到启动文件 app.js
      可以看到 /bin/ast 是个软链接, 在bin目录下通过 ls -li ast 查看其对应的执行文件 /usr/local/lib/node_modules/astroboy-cli/bin/ast.js

    3. 分析执行文件
      require('../index.js') 即 /astroboy-cli/index.js
      这个文件的作用是加载 plugins, 其中需要特别关注 astroboy-cli/src/plugins/dev, 其作用是开启服务
      注册并执行 plugin.action, 这部分参考 commander 的文档 https://github.com/tj/commander.js/blob/master/Readme_zh-CN.md
      分析 astroboy-cli/src/plugins/dev/action.js 文件, 其会寻找 ${projectRoot}/app/app.(js|ts) 作为启动文件
      projectRoot 取 process.cwd(), 可以看到如果要用 app.ts 作为启动文件, 需要加 --ts 配置, 这解释了步骤2中找不到 app.js 的问题
      执行项目入口文件 guang-admin-node/app/app.ts
      app.ts 中 创建了 YouzanFramework 实例, 其继承了 Astroboy, 并且因为没有自己的构造函数, 执行了 Astroboy 的执行函数
      至此, 调试环境打通, 后续的调试可以直接在 Astroboy 的构造函数中打断点

  {
    "version": "0.2.0",
    "configurations": [
      {
        "type": "node",
        "request": "launch",
        "name": "dev",
        "cwd": "${workspaceFolder}/",
        "program": "/usr/local/lib/node_modules/astroboy-cli/bin/ast.js",
        "args": [
          "dev", "--mock", "--env", "qa", "--ts", "--inspect=8888", "--tsconfig", "tsconfig.json"
        ],
        "env": {
          "NODE_ENV":"development",
        },
        "autoAttachChildProcesses": true,
        "stopOnEntry": true
      }
    ]
  }
