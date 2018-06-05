---
title: "Vim"
date: 2018-04-24T13:37:56+08:00
draft: true
---
### 替换

[range]s[ubstitute]/{pattern}/{string}/[flags] [count]，1,20为1到20行，%为整个文件，同1,$。如，%s/\(^.*\)\(value.*\)\(\ wid.*>\)\(.*\)\(<\/input>\)/<option\ \2>\4<\/option>/g

[range]g[lobal]/{pattern}/[cmd]

如何找出并删除文档中重复的行，可以sort file|uniq > new，但如何用vim实现

### 设置
- :set nu "显示行号。
- :set nonu "不显示行号。
- xmap <C-k> :m '< -- <CR> gv "将选中部分诸行向上移动
- xmap <C-k> :m '> + <CR> gv "将选中部分诸行向下移动
- set list listchars=eol:$,tab:>-,trail:~,extends:>,precedes:< "set nolist回归正常
- set listchars+=space:␣

### viml


### plugin

### python 

```java
fun! Foo()
python << eof
import vim

cur_buf = vim.current.buffer
print "Lines: {0}".format(len(cur_buf))
print "Contents: {0}".format(cur_buf[-1])

class Demo:
    def __init__(self):
        print 'foo demo'
Demo()
eof
endfun

:call Foo()
```

### ref
- [spf](https://github.com/spf13/spf13-vim)
