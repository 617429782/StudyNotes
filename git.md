<h2>git 简介</h2>
  <h3>四个分区: 工作区、暂存区(索引 .git/index)、本地仓库、远程仓库</h3>
    <table width="50%"><tr><td bgcolor=#fff><img src="https://img.yzcdn.cn/upload_files/2020/12/18/FlWMWzIX9WE7PW-7eyeq8uaEJ_3p.png"></td></tr></table>
 
  在 '工作区' 新增、修改、删除 文件, 对应 vscode 的 '更改'
  通过 git add 将'工作区'的文件添加到本地仓库, 将产生一个 blob 类型的节点, 对应 vscode 的 '暂存的更改'


  <h3>目录结构</h3>

    .git
      ├── FETCH_HEAD
      ├── HEAD
      ├── config
      ├── description
      ├── hooks
      │   ├── applypatch-msg.sample
      │   ├── ...
      ├── index # 索引信息
      ├── info
      │   └── exclude
      ├── objects
      │   ├── d8
      │   │   └── 00886d9c86731ae5c4a62b0b77c437015e00d2
      │   ├── e6
      │   │   └── 9de29bb2d1d6434b8b29ae775ad8c2e48c5391
      │   ├── info
      │   └── pack
      └── refs
          ├── heads
          └── tags

    .git/hooks
      钩子函数, 在触发特定事件(如commit、merge、receive)时增加处理
      git仓库建立后 hooks 下会有一些 xxx.sample 的示例钩子, 去掉 .sample 即可生效, 即: 文件名与钩子名相同
      git 支持的钩子列表 https://git-scm.com/docs/githooks
      钩子默认用 Shell Script 编写, 也可以用 shell 调用 node 脚本, 我们的小程序里用的 husky 作为管理工具
        
    .git/index
      索引信息, 是一个二进制文件, 

    .git/objects
      所有的节点, 包四种类型节点: blob、tree、commit、tag(标签节点, 不常用)
        blob 节点: 
          基于 '文件内容', 通过 SHA1 算法得到的 哈希值 
          哈希算法, 按固定规则(只要源内容不变, 每次的结果都是相同的) 将源字符串映射成 长度固定、不可逆的目标字符串, 常用于密码存储、文件完整性校验等场景
          内容相同的 多个文件, 只生成 一个blob节点
        tree 节点:
          
        commit 节点:
        tag 节点: 
      查看节点的命令
        tree .git/objects           # 查看所有节点
        git cat-file -t [file]      # 查看节点类型
        git cat-file -p [file]      # 查看节点内容
        git ls-files --stage        # 查看仓库中的所有 文件 与其所对应的 节点

    .git/config
      core.ignorecase = true  # 忽略大小写, 默认为true, 修改文件名大小写 不会修改索引
      # git config 命令, 不给值为读, 给值为写
      git config [--system(系统级) / global(用户级) / local(项目级) / file(文件级)] core.ignorecase [true/false]
      

<h2>git 常用命令 与 分析</h2>
  <h3>创建型</h3>
    git init # 在本地初始化一个空的git仓库, 生成 .git 文件
    git clone [url] # clone 命令是从远程获取了 完整的版本库
    
  <h3>开发型</h3>
    工作区 & 暂存区
      # 添加 工作区文件 到 暂存区(在vscode中对应 暂存的修改), 在 .git/objects 生成一个 blob类型 的节点
      # 被添加到 暂存区 到文件会被 追踪, 之后对该文件的 首次修改 将会另外生成一个 blob类型 的节点
      git add
        # 暂存对'工作区'文件的修改, 在'本地仓库'中生成 对应的blob节点
        # 更新索引,  
        git add [file1] [file2] ... 
        git add [dir]
        git add .
        git add -p [file1]
      git rm  
        git rm [file1] [file2] ...  # 删除工作区的文件, 将 删除操作 存入暂存区
        git rm --cached [file] # 取消对文件的追踪, 删除其索引
      git mv
        git mv [file-original] [file-renamed] # 文件改名, 存入暂存区

    git commit        # 将 '索引' 提交到 '本地仓库' 中

    git push          # 将 '本地仓库' 内容提交到 '远程仓库' 中
    git pull
    git checkout

  <h3>协作型</h3>
    git fetch         # 从远程仓库获取 某个分支的最新版本, 但不与本地合并, fetch + merge = pull
    git merge
    git rebase
    git blame dirPath -L startLine,endLine    # 查看某几行代码的最近提交者 (vscode 可以装 gitlens 插件)
    git cherry-pick <commitHash>  # 将指定的 commit 合并进当前分支

<h2>git 常用操作 与 分析</h2>
  设置SSH
    作用: 
    教程: https://www.cnblogs.com/ayseeing/p/3572582.html

  github 访问加速 https://cloud.tencent.com/developer/article/1426844
    github 访问慢的一个原因是服务器在海外, dns耗时较长
    可以修改本地 hosts 文件 /etc/hosts (需要sudo), 就近指定ip, 跳过dns解析
    网络寻址工具 http://tool.chinaz.com/dns/

  设置忽略文件
    设置 .gitkeep
      空的文件夹不会被 git 追踪, 如果有需要, 比如 output 文件夹, 可以在文件下添加一个 .gitkeep 文件
    设置 .gitignore

  分支管理

  合并 commit

  查看历史版本
    log
    show
    reflog: 相比 log，reflog 可以获取到已被删除的 commit 记录
    rev-list: 时间倒序输出指定条件的 commit 记录 https://cloud.tencent.com/developer/section/1138780

    场景: 
    操作: 

  回滚
    场景: 代码发布后线上出现问题, 重新发版太慢, 需要临时返回到代码的上一版本
    操作:

  从整个历史中删除一个文件 xxx.txt
    场景: 代码将要共享, 但是发现某个文件包含敏感信息, 比如 密码
    操作: 
      git filter-branch --tree-filter 'rm -f xxx.txt' HEAD
      通知所有开发者, 弃用之前迁出的分支, 从新分支获取开发分支

  忽略已有文件
    git rm --cached filename / git rm --cached -r dirname (暂存删除操作, 但不会影响工作区的文件)
    在 .gitignore 中写入被删除的文件/目录

  合并一个分支上某一次的修改到另一个分支上
    场景: 不小心在 merge 上直接修改了代码, 想在 feature 分支上同步修改 (不能用merge命令, merge分支上可能有废代码)
    操作: 
      提交 merge 的修改, 找到 commitID

## 常用命令
  - 删除本地分支
    git branch -d <BranchName>

  - 撤销某次提交, 撤销操作需作为新的 commit 提交
    git revert <commitId>
  
## 命令详解



## pro git
  分支 是一个指针，指向 某一系列 commit 对象中的 首个
  HEAD 指针永远指向当前所处的 分支，即 git branch 列出带 * 的那个，可以理解成 当前分支的别名

  commit 对象的产生时机
    git commit 命令执行时，会在当前 HEAD所处分支 指向的 commit 对象之上，创建一个新的 commit 对象，HEAD所处分支 跟随指向新对象

  一个 commit 对象包含
    tree 对象：文件的索引信息
    若干个 blob 对象：每个文件内容对应的 blob 对象
    其他提交元信息，比如 提交者、提交时间 等等