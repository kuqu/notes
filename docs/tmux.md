
# Tmux使用总结

{docsify-updated}

## 概念

tmux 是一个终端窗口管理器。一般ssh远程连接的时候用的比较多，可以保证连接意外中断后程序不退出。

tmux 有三个层次的概念：session，window，pane。session 中可以包括多个window，window中可以包括多个pane。常用的操作就是对三个概念的操作。

tmux 的快捷键默认采用前缀键ctrl+b。


## 安装

```
apt install tmux
```

## 帮助

```
ctrl+b, ? 
tmux list-keys
tmux list-commands
```
## Session

创建
```
tmux
tmux new -s <session_name>
```

查看session
```
tmux ls
tmux list-session
```

进入
```
tmux attach -t 0
tmux attach -t <session-name>
```

切换
```
tmux switch -t 0
tmux switch -t <session-name>
Ctrl+b, s
```

分离
```
tmux detach
ctrl+b, d
```

杀死
```
tmux kill-session -t 0
tmux kill-session -t <session-name>
```

重命名
```
tmux rename-session -t 0 <new-name>
ctrl+b, $
```

## Window

创建
```
tmux new-window
tmux new-window -n <window-name>
tmux+b, c
```

切换
```
tmux select-window -t 0
tmux select-window -t <window-name>
ctrl+b, p/n 
ctrl+b, <number> 切换到编号window
ctrl+b, w 列表切换
```

关闭
```
tmux kill-window
```

重命名
```
tmux rename-window <new-name>
ctrl+b, , 
```

## Pane


创建
```
tmux split-window
tmux split-window -h
ctrl+b, %
ctrl+b, "
```

移动光标
```
tmux select-pane -U
tmux select-pane -D
tmux select-pane -L
tmux select-pane -R
ctrl+b, <arrow-key>
ctrl+b, ; 上一个Pane
ctrl+b, o 下一个Pane
```

移动Pane
```
tmux swap-pane -U
tmux swap-pane -D
tmux swap-pane -L
tmux swap-pane -R
```

调整大小
```
ctrl+b, ctrl+<arrow-key>
ctrl+b, z 全屏，再次使用则恢复
ctrl+b, ! Pane升级为独立window
```

退出
```
exit
ctrl+d
ctrl+b, x
```




