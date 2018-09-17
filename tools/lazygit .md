# lazygit 

[lazygit](https://github.com/jesseduffield/lazygit)一个git管理工具，喜好用终端管理工程代码的朋友，可以尝试。其他可视化工具可以用[SourceTree](https://www.sourcetreeapp.com/)。

官网地址: [GITHUB](https://github.com/jesseduffield/lazygit)

## 特性/优势

* 轻松添加文件
* merge冲突处理
* 轻松切换分支
* 浏览diffs(branch、commits、stash)
* 快速push/pull
* 压缩/修改commit

## 如何使用

### 安装(Homebrew方式)

```
brew tap jesseduffield/lazygit
brew install lazygit
```

其他方式参考官网。

### 常用快捷键

* 全局

<pre>
  <kbd>←</kbd><kbd>→</kbd><kbd>↑</kbd><kbd>↓</kbd>/<kbd>h</kbd><kbd>j</kbd><kbd>k</kbd><kbd>l</kbd>:               切换模块(导航)
  <kbd>PgUp</kbd>/<kbd>PgDn</kbd> or <kbd>ctrl</kbd>+<kbd>u</kbd>/<kbd>ctrl</kbd>+<kbd>d</kbd>:   浏览`diff`面板
                                     (for <kbd>PgUp</kbd> and <kbd>PgDn</kbd>, use <kbd>fn</kbd>+<kbd>up</kbd>/<kbd>fn</kbd>+<kbd>down</kbd> on osx)
  <kbd>q</kbd>:                                退出
  <kbd>p</kbd>:                                pull
  <kbd>shift</kbd>+<kbd>P</kbd>:                         push
</pre>

* 状态面板

<pre>
  <kbd>e</kbd>:        编译配置信息
  <kbd>o</kbd>:        打开配置信息
</pre>

* 文件面板

<pre>
  <kbd>space</kbd>:    文件的暂存状态切换
  <kbd>a</kbd>:        所有文件暂存/不暂存
  <kbd>c</kbd>:        提交
  <kbd>shift</kbd>+<kbd>C</kbd>: 使用编辑器提交
  <kbd>shift</kbd>+<kbd>S</kbd>: 储蓄文件
  <kbd>t</kbd>:        添加补丁 (i.e. pick chunks of a file to add)
  <kbd>o</kbd>:        打开
  <kbd>e</kbd>:        编辑
  <kbd>s</kbd>:        sublime方式打开 (requires 'subl' command)
  <kbd>v</kbd>:        vscode方式打开 (requires 'code' command)
  <kbd>i</kbd>:        添加到.gitignore
  <kbd>d</kbd>:        删除没有tracked的文件/ checkout tracked的文件
  <kbd>shift</kbd>+<kbd>R</kbd>: 刷新文件
  <kbd>shift</kbd>+<kbd>A</kbd>: 终止merge
</pre>

* 分支面板

<pre>
  <kbd>space</kbd>:   切换分支
  <kbd>f</kbd>:       强制切换分支
  <kbd>m</kbd>:       merge到当前打开分支
  <kbd>c</kbd>:       输入分支名称方式checkout
  <kbd>n</kbd>:       新建分支
  <kbd>d</kbd>:       删除分支
  <kbd>D</kbd>:       强制删除分支
</pre>

* Commits面板

<pre>
  <kbd>s</kbd>:       压缩commits (仅对第一个commit有效)
  <kbd>r</kbd>:       commit重命名
  <kbd>shift</kbd>+<kbd>R</kbd>: 使用编辑器重命名commit
  <kbd>g</kbd>:       重置到某个commit
</pre>

* 储蓄面板

<pre>
  <kbd>space</kbd>:   应用
  <kbd>g</kbd>:       推出
  <kbd>d</kbd>:       删除
</pre>

* 弹出面板

<pre>
  <kbd>esc</kbd>:     关闭/取消
  <kbd>enter</kbd>:   确认
  <kbd>tab</kbd>:     换行 (编译状态下)
</pre>

* 处理合并冲突(Diff面板zz)

<pre>
  <kbd>←</kbd><kbd>→</kbd>/<kbd>h</kbd><kbd>l</kbd>: 导航/移动
  <kbd>↑</kbd><kbd>↓</kbd>/<kbd>k</kbd><kbd>j</kbd>: 选择大块
  <kbd>space</kbd>:      选择某块冲突
  <kbd>b</kbd>:         选择全部
  <kbd>z</kbd>:         回撤 (only available while still inside diff panel)
</pre>

## 其他

视屏教程参考: [here](https://www.youtube.com/watch?v=VDXvbHZYeKY)

