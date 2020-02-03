# MacOS 原生终端优化

## 个人需求

Mac上的终端产品俨然已经是iTerm2大一统。但是对于我个人，曾经在windows上安装babun，发现zsh那些强大的功能我都没用过。
而在Linux各个发行版上摸爬滚打这些年，常常简单的配置下shell就能满足我对于Shell的全部需要。于是打开用户目录下的bash配置（vi ~/.bash_profile）直接折腾Mac的原生Terminal。

## 设置命令提示符

![prompt](https://raw.githubusercontent.com/progenghis/p/res/macos-terminal-optimize/prompt.png)

Mac终端默认的命令提示符除了当前目录，都是些没用的东西。修改其内容可以让提示符更加简洁高效。其对应修改环境变量PS1。

```shell
export PS1="\[\033[33;1m\]\W\[\033[32m\]\[\033[m\] \$ "
```

*PS1*支持的转义字符包括：

-  \u 用户名
-  \h 主机名第一部分
-  \H 主机名全称
-  \w 当前工作目录（如 “/home/username/mywork”）
-  \W 当前工作目录的“基名 (basename)”（如 “mywork”)
-  \t 24 小时制时间
-  \T 12 小时制时间
-  \@ 带有 am/pm 的 12 小时制时间
-  \d “Sat Dec 18″ 格式的日期
-  \s shell 的名称（如 “bash”)
-  \v bash 的版本（如 2.04）
-  \V Bash 版本（包括补丁级别）
-  \n 换行符
-  \r 回车符
-  \\ 反斜杠
-  \a ASCII 响铃字符（也可以键入 07）
-  \e ASCII 转义字符（也可以键入 33)
-  \[ 这个序列应该出现在不移动光标的字符序列（如颜色转义序列）之前。它使 bash 能够正确计算自动换行。
-  \] 这个序列应该出现在非打印字符序列之后。

其次还能够通过编码设置具体的颜色样式，其由分号分隔的两个数值决定，分号前是颜色码，分号后是样式码。
样式码包括：

- 0 关闭所有属性  
- 1 设置高亮度  
- 4 下划线  
- 5 闪烁  
- 7 反显  
- 8 消隐
- m 意味着设置属性然后结束非常规字符序列

颜色码包括：

- 30 black  
- 31 red  
- 32 green  
- 33 yellow  
- 34 blue  
- 35 purple  
- 36 cyan  
- 37 white

这些值将设置前景色，如果需要设置背景色则使用40～47。

## 上色

给黑白的终端增添彩色是优化的初步。
上色分为两步。
第一步就是去Terminal的Preferences中选一个可心的主题（Profiles）。这里的主题可能会对后面设置的颜色有影响，推荐[IR_Black](https://github.com/justincase/IR_Black-OSX)。
至于第二步，则是开启Shell的彩色开关。这些开关大都是由环境变量和命令参数控制的。

### ls命令

![ls-color](https://raw.githubusercontent.com/progenghis/p/res/macos-terminal-optimize/ls-color.png)

ls是Shell中最最常用的命令，在Mac中，CLICOLOR控制其是否展示颜色，LSCOLORS则设置具体颜色值。

```shell
export CLICOLOR=1
export LSCOLORS=ExFxBxDxCxegedobagacad
```
LSCOLORS按照顺序每两个字母一组，共11组，分别对应以下项目的颜色：

- directory
- symbolic link
- socket
- pipe
- executable
- block special
- character special
- executable with setuid bit set
- executable with setgid bit set
- directory writable to others, with sticky bit
- directory writable to others, without sticky bit

各个字母代表的颜色和样式如下：

- a黑色
- b红色
- c绿色
- d棕色
- e蓝色
- f洋红色
- g青色
- h浅灰色
- A黑色粗体
- B红色粗体
- C绿色粗体
- D棕色粗体
- E蓝色粗体
- F洋红色粗体
- G青色粗体
- H浅灰色粗体
- x系统默认颜色

### grep命令

![grep-color](https://raw.githubusercontent.com/progenghis/p/res/macos-terminal-optimize/grep-color.png)

另一个常用命令grep也对颜色敏感，我们希望它能高亮显示匹配的字符，方便查找。通过环境变量GREP_OPTIONS和GREP_COLOR可以为其设置参数。

```shell
export GREP_OPTONS='--color=auto'
export GREP_COLOR='1;32'
```

*--color*本身有3个值可选，分别是

- never 从不添加颜色
- always 总是添加颜色，当通过管道或重定向时就会多出一些控制字符
- auto 仅当输出到控制台时才添加颜色

具体的颜色样式由分号分隔的两个数值决定，分号前是样式码，分号后是颜色码。取值与命令提示符中相同。

### vim命令

![vim-color](https://raw.githubusercontent.com/progenghis/p/res/macos-terminal-optimize/vim-color.png)

Vim在Mac中本身是有配色方案的，不知为什么默认不启用。其实仅需要修改`～/.vimrc`，增加`syntax on`这一项配置即可。
如果对默认的配色主题不满意，可以再增加一项`colorscheme <theme>`来设置配色主题，可选的主题都在`/usr/share/vim/vim80/colors`中。

## 显示Git状态

![git-status](https://raw.githubusercontent.com/progenghis/p/res/macos-terminal-optimize/git-status.png)

终日与代码为伴，Git状态自然是需要时时关注的。这里希望显示当前所处分支及是否发生修改。
这里需要用到一些Git命令，并将其结果呈现在命令行提示符中：

```shell
# add git information to prompt
function parse_git_dirty {
    [[ -z $(git status --porcelain 2> /dev/null) ]] || echo " *"
}

function parse_git_branch {
    git branch 2> /dev/null | sed -e '/^[^*]/d' -e "s/* \(.*\)/ (\1$(parse_git_dirty))/"
}

export PS1="\[\033[33;1m\]\W\[\033[32m\]\$(parse_git_branch)\[\033[m\] \$ "
```

## 常用命令Alias

![alias](https://raw.githubusercontent.com/progenghis/p/res/macos-terminal-optimize/alias.png)

一些常用的命令在原生终端中并未设置别名，用起来有点麻烦。比如查看详细文件列表需要用`ls -la`，返回上一级菜单需要`cd ..`。于是增加了一些常用的alias：

```shell
alias ls="ls -GFh"
alias ll="ls -la"
alias l=ll
alias cd..="cd .."
alias ..="cd .."
```

## 自建指令库

最后，还有一些命令，比如登录到远程主机、连接AWS等，统统自己写脚本，放在`~/Scripts`中。并且添加到`PATH`环境变量中，随时可以调用。

```shell
export PATH=$PATH:~/Scripts
```


> Written with [StackEdit](https://stackedit.io/) at Dec 4, 2018.
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTA1ODM4NzkyXX0=
-->