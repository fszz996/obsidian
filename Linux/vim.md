
vim模式：
![[Pasted image 20260720000123.png]]

### 插入模式

a：光标后
大A：行尾
I：行头
小o：下一行
大O：上一行、



![[Pasted image 20260720000847.png]]

### 尾行模式

：q 退出
：wq 保存并退出

显示行号：
：set nu
：set nonu

：50   -跳转到第50行

#### 替换
：n1，n2s /hello/hi/g
	|		                 |
要替换的行			  global

### 命令模式

^:行首
$:行尾

yy：复制一行
p：粘贴
dd：剪切

G:最后一行
gg：第一行

#### 查找：
/  向下查找
? 向上查找
n next one
N latter one

u 撤销
