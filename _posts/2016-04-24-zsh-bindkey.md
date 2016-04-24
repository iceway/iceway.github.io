---
title: zsh快捷键
layout: post
date: 2016-04-24
tags: zsh bindkey
categories: 效率工具
---

zsh使用`bindkey`作为快捷键绑定的工具，关于`bindkey`的用于，可以查看手册`man zshzle`。


## 查看当前绑定的快捷键

不跟参数执行`bindkey`会列出当前绑定的快捷键，如下（输出内容的含义参见后面的快捷键设定部分）：

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
"^M" accept-line
"^N" down-line-or-history
"^O" accept-line-and-down-history
"^P" up-line-or-history
"^Q" push-line
"^R" history-incremental-search-backward
"^S" history-incremental-search-forward
"^T" transpose-chars
"^U" kill-whole-line
"^V" quoted-insert
"^W" backward-kill-word
"^X^B" vi-match-bracket
"^X^E" edit-command-line
"^X^F" vi-find-next-char
"^X^J" vi-join
"^X^K" kill-buffer
"^X^N" infer-next-history
"^X^O" overwrite-mode
"^X^R" _read_comp
"^X^U" undo
"^X^V" vi-cmd-mode
"^X^X" exchange-point-and-mark
"^X*" expand-word
"^X=" what-cursor-position
"^X?" _complete_debug
"^XC" _correct_filename
"^XG" list-expand
"^Xa" _expand_alias
"^Xc" _correct_word
"^Xd" _list_expansions
"^Xe" _expand_word
"^Xg" list-expand
"^Xh" _complete_help
"^Xm" _most_recent_file
"^Xn" _next_tags
"^Xr" history-incremental-search-backward
"^Xs" history-incremental-search-forward
"^Xt" _complete_tag
"^Xu" undo
"^X~" _bash_list-choices
"^Y" yank
"^[^D" list-choices
"^[^G" send-break
"^[^H" backward-kill-word
"^[^I" self-insert-unmeta
"^[^J" self-insert-unmeta
"^[^L" clear-screen
"^[^M" self-insert-unmeta
"^[^[[A" up-history
"^[^[[B" down-history
"^[^[[C" forward-word
"^[^[[D" backward-word
"^[^_" copy-prev-word
"^[ " magic-space
"^[!" expand-history
"^[\"" quote-region
"^[\$" spell-word
"^['" quote-line
"^[," _history-complete-newer
"^[-" neg-argument
"^[." insert-last-word
"^[/" _history-complete-older
"^[0" digit-argument
"^[1" digit-argument
"^[2" digit-argument
"^[3" digit-argument
"^[4" digit-argument
"^[5" digit-argument
"^[6" digit-argument
"^[7" digit-argument
"^[8" digit-argument
"^[9" digit-argument
"^[<" beginning-of-buffer-or-history
"^[>" end-of-buffer-or-history
"^[?" which-command
"^[A" accept-and-hold
"^[B" backward-word
"^[C" capitalize-word
"^[D" kill-word
"^[F" forward-word
"^[G" get-line
"^[H" run-help
"^[L" down-case-word
"^[N" history-search-forward
"^[OA" up-line-or-beginning-search
"^[OB" down-line-or-beginning-search
"^[OC" forward-char
"^[OD" backward-char
"^[P" history-search-backward
"^[Q" push-line
"^[S" spell-word
"^[T" transpose-words
"^[U" up-case-word
"^[W" copy-region-as-kill
"^[[1;5C" forward-word
"^[[1;5D" backward-word
"^[[1~" beginning-of-line
"^[[2~" yank
"^[[3~" delete-char
"^[[4~" end-of-line
"^[[5~" up-line-or-history
"^[[6~" down-line-or-history
"^[[A" up-history
"^[[B" down-history
"^[[C" forward-char
"^[[D" backward-char
"^[[Z" reverse-menu-complete
"^[_" insert-last-word
"^[a" accept-and-hold
"^[b" backward-word
"^[c" capitalize-word
"^[d" kill-word
"^[f" forward-word
"^[g" get-line
"^[h" run-help
"^[l" "ls^J"
"^[m" copy-prev-shell-word
"^[n" history-search-forward
"^[p" history-search-backward
"^[q" push-line
"^[s" spell-word
"^[t" transpose-words
"^[u" up-case-word
"^[w" kill-region
"^[x" execute-named-cmd
"^[y" yank-pop
"^[z" execute-last-named-cmd
"^[|" vi-goto-column
"^[~" _bash_complete-word
"^[^?" backward-kill-word
"^\^[[A" up-history
"^\^[[B" down-history
"^\^[[C" forward-char
"^\^[[D" backward-char
"^_" undo
" " magic-space
"!"-"~" self-insert
"^?" backward-delete-char
"\M-^@"-"\M-^?" self-insert
➜
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
