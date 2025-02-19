---
title: xv6-lab1-TAB completion
---
<h1 >关于XV6 Lab1的shell tab completion challenge</h1>
首先得想办法让terminal能够不回显tab<br>
其次是参考GNU readline library，其中有非常成熟的命令补全以及命令历史记录的实现<br>
实际上，很多的shell、一些工具的命令行模式，都使用了这个库<br>
<h2>关于shell tab completion</h2>
目前的terminal driver似乎只能支持行缓冲模式，即，让read返回直到用户键入回车前的所有字符<br>
参照readline的实现，我们现在想要一个这样的模式：每次键入一个按键，read就马上返回这个按键对应的编码值<br>
这样一来，针对返回的一般字母数字以及回车，我们将其回显，而tab则不回显，返回的字符用一个buffer缓冲<br>
当返回的字符是tab时，shell可以进行completion尝试补全缓冲区的字符串，这里可以基于当前目录的文件名进行补全