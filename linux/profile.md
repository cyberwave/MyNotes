# 一、配置
1. alias 配置终端命令
可以把一些经常使用的或敲入比较复杂的命令起个别名这样输入简短的命令就可以实现同样的效果。
注意以下 `alias` 后面等号两侧不要有空格
```shell
# 大部分终端配置文件为 .profile, 有的可能是 .zshrc或其他
vi .profile 
alias ls='ls -l'
```

或都使用简短的命令代替复杂的命令
```shell
alias cdp='cd ~/profile'
```

在配置中还可以使用变量以及函数
```shell
alias cdl='func(){cd $1;ls -l;};func'
```
$1为cdl后面的第一个参数，上面鸽子实现的功能是切换到相应的目录后，再列出目录下面所有文件

# 二、解开 .deb 文件
假设文件名为 hello.deb
```shell
ar vx hello.deb
```
会得到三个文件
```shell
debian-binary
control.tar.gz
data.tar.gz
```
使用 `tar` 解开 data.tar.gz 即可得到 deb 文件中的数据文件