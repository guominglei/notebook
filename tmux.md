# tmux

## 会话管理
    1、新建一个会话
    tmux new-session -s  会话名称
    tmux new -s  会话名称

    2、detach 会话。让进程后台运行。
    快捷键 ctrl + b + d

    3、列出当前有多少会话
    tmux list-sessions
    tmux ls

    3、attach 会话
    tmux attach-session -t 会话名称
    tmux attach -t 会话名称
    tmux a -t 会话名称

    快捷键 ctrl + b 是快捷键前缀

    ctrl + b + c 创建一个新的窗口
    删除当前回话  Ctrl+b 输入 :kill-session
    删除所有会话  Ctrl+b 输入 :kill-server

## 常用命令
    
