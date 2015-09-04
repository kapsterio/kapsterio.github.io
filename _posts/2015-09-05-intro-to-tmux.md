# TMUX的一点总结

## 工作原理

tmux是一种典型的client－server架构的程序，session通过每个client显示在终端上，而所有session是通过一个单独的server来管理的，server和每个client都是单独的进程，它们之间通过一个socket来通信。

tmux中通过各种命令来管理server和所连接的client，参见tmux的manpage，在tmux之外可以直接在shell中输入$tmux command 来运行这些命令。当然在tmux中（处于某个client里面时）也可以运行命令，有两种方式：

 - 通过快捷键来发送相应预先绑定到某些key的命令
 - 通过键入prefix+:来进行命令行输入相应的命令

tmux的配置文件也是由一堆命令tmux命令构成的，运行tmux时会首先查找/etc/.tmux.conf文件加载初始配置，如果没有则找~/.tmux.conf文件来加载，后者会覆盖掉前者
