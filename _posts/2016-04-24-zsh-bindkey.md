---
title: zsh快捷键
layout: post
date: 2016-04-24
tags: zsh bindkey
categories: 效率工具
---

zsh使用`bindkey`作为快捷键绑定的工具，关于`bindkey`的用于，可以查看手册`man zshzle`。


## 查看当前绑定的快捷键

不跟参数执行`bindkey`会列出当前绑定的快捷键。下面列出了bindkey的部分输出（输出内容的含义参见后面的快捷键设定部分）：

```bash
➜  snippets git:(master) bindkey
"^@" set-mark-command
"^A" beginning-of-line
"^B" backward-char
"^D" delete-char-or-list
"^E" end-of-line
"^F" forward-char
"^G" send-break
"^H" backward-delete-char
"^I" expand-or-complete
"^J" accept-line
"^K" kill-line
"^L" clear-screen
```

## 常用快捷键的默认绑定

常用的默认快捷键如下表（针对emac模式）：

| 快捷键	| 功能	|
|-------|-------|
| `Ctrl-f`	| 向前移动一个字符	|
| `Ctrl-b`	| 向后移动一个字符	|
| `Alt-f`	| 向前移动一个词	|
| `Alt-b`	| 向后移动一个词	|
| `Ctrl-a`	| 移动到行首		|
| `Ctrl-e`	| 移动到行尾		|
| `Ctrl-h`	| 删除光标前一个字符	|
| `Ctrl-d`	| 删除光标所在处的字符	|
| `Ctrl-w`	| 从光标处向前删除一个词		|
| `Ctrl-u`	| 剪切行首到光标的内容	|
| `Ctrl-k`	| 剪切光标到行尾的内容	|
| `Ctrl-y`	| 粘贴剪切的内容	|
| `Ctrl-l`	| 清屏，相当于命令`clear`	|
| `Alt-'`	| 将本行加上引号	| 
| `Ctrl-x,u`	| 撤销上一步改动	|
| `Ctrl-_`	| 撤销上一步改动	|
| `Ctrl-t`	| 交换光标前后的字符	|
| `Alt-t`	| 交换光标前后的词	|
| `Alt-0`~`Alt-9` | 粘贴上一个命令的第几个参数	|
| `Alt-.`	| 粘贴上一个命令的最后一个参数	|
| `Ctrl-x,Ctrl-e` | 打开编辑器编辑当前命令行内容	|
| `Ctrl-r`	| 向前搜索命令历史	|
| `Ctrl-s`	| 向后搜索命令历史	|
| `Ctrl-n`	| 历史命令的下一条	|
| `Ctrl-p`	| 历史命令的上一条	|
| `Alt-h`	| 显示命令行中命令（第一个词）的手册	|
| `Alt-?`	| 显示命令行中命令（地一个词）的绝对路径	|

## 快捷键设定

可以在zsh配置文件`~/.zshrc`中通过`bindkey`命令自定义快捷键。

`bindkey`的第一个参数就是要指定的快捷键（使用对应的[CSI][CSI]序列），第二个参数就是快捷键实现的功能（`man zshzel`可以查看支持的功能）。

[CSI]: http://en.wikipedia.org/wiki/ANSI_escape_code

比如：

```bash
bindkey '^[^[[C' emacs-forward-word
```

快捷键是'^[^[[C'（对应的就是Alt和右方向键的组合），而实现功能是emacs模式下向前移动一个词。在命令行上先按`Ctrl-v`，然后再按组合键，可以的到该组合键的CSI序列。

在命令行上先按`Ctrl-v`，然后再按组合键，可以的到该组合键的CSI序列。
