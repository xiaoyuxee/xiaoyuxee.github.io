---
title: Mac下高效地使用Eclipse
date: 2016-01-17 19:04:43
tags: Eclipse
categories:
toc: true
---

{% blockquote 孔子，论语·卫灵公 %}
工欲善其事，必先利其器。
{% endblockquote %}

[Eclipse](http://www.eclipse.org/)想必是很多同学每天接触的IDE，那么熟练使用eclipse中的快捷键，对每天coding效率的提升将会带来显著的提示。

<!-- more -->

## 文件与工程管理
`command`+`N`：新建文件/工程
`command`+`shift`+`R`：快速打开文件，支持`?`、`*`匹配
`command`+`I`：打开文件属性
`option`+`I`：打开文件属性
`command`+`S`：保存当前文件
`command`+`shift`+`S`：保存所有文件
`command`+`W`：关闭当前文件
`command`+`shift`+`W`：关闭所有文件
`F5`：从本地文件系统刷新资源

## 编辑窗口
`command`+`shift`+`[`：返回上一个编辑的文件
`command`+`shift`+`]`：返回下一个编辑的文件
`control`+`M`：最大化/最小化当前窗口
`command`+`E`：在打开的文件列表中切换
`command`+`L`：跳至指定行

## 编辑文件
`command`+`C`：复制
`command`+`X`：剪切
`command`+`V`：黏贴
`command`+`Z`：取消上步操作
`command`+`shift`+`Z`：重新执行上步被取消的操作
`command`+`D`：删除当前这一行
`option`+`Arrow Up`/`Arrow Down`：向上/向下*移动*当前行或者当前选择的区块
`command`+`option`+`Arrow Up`/`Arrow Down`：向上/向下*复制*当前行或者当前选择的区块
`command`+`delete`：删除上一个单词
`shift`+`enter`：在下一行插入编辑
`command`+`shift`+`enter`：在上一行插入编辑
`command`+`shift`+`X`：将选择区块全部变为大写
`command`+`shift`+`Y`：将选择区块全部变为小写

## 搜索和替换
`command`+`F`：打开*查找和替换*对话框
`control`+`H`：在工作区间中搜索（java、task、file等）
`control`+`J`：输入内容后，*正向*查找字符串
`command`+`shift`+`J`：输入内容后，*反向*查找字符串

## 代码编辑
`command`+`I`：修正当前选择区块的缩进
`command`+`shift`+`F`：格式化代码
`command`+`/`：注释/取消注释当前行
`command`+`control`+`/`：添加块注释`/* ... */`
`command`+`control`+`\`：取消块注释
`command`+`option`+`J`：增加元素注释`/** ... *`
`command`+`1`：快速修复
`command`+`shift`+`M`：增加`import`
`command`+`shift`+`O`：组织包`import`
`option`+`/`：代码提示

## 代码信息
`command`+`O`：快速查看代码结构
`command`+`T`：快速查看类型继承
`command`+`shift`+`T`：搜索查看类
`F2`：查看类、方法、变量注释说明
`F3`：跳至类、方法、变量声明处
`F4`：在类型继承窗口（视图）中打开
`control`+`option`+`H`：查看类、方法调用层次
`command`+`shift`+`U`：在当前类中查找表达式出现的地方
`command`+move over method：打开方法声明或则实现

## 重构
`control`+`option`+`R`：类、方法、变量等重命名
`control`+`option`+`V`：方法移动
`control`+`option`+`M`：抽取方法
`control`+`option`+`L`：抽取本地变量
`control`+`option`+`I`：内联收拢变量

## 调试与运行
`command`+`F11`：Run
`shift`+`command`+`B`：设置断点
`F5`：单步进入
`F6`：下一行
`F7`：跳出，进行下一步
`F8`：调至下一个断点

## 参考资料

[Eclipse Shortcuts ](http://www.shortcutworld.com/en/win/Eclipse.html)
[玩转Eclipse — 快捷键设置及汇总](http://blog.csdn.net/jmyue/article/details/16960199)
[Working Efficiently with JUnit in Eclipse](http://www.codeaffine.com/2012/11/26/working-efficiently-with-junit-in-eclipse-2/)

