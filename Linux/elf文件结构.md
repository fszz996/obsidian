可以用`readelf`命令来查看elf文件

elf是linux系统下的可执行文件,里面存储了:代码,数据,文件头,程序头.节头等
就用最常用的/bin/ls这个文件来示例吧

# 文件头
记录了一些程序的基本信息

```
fszz@fszz:~/ctf$ readelf -h /bin/ls
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              DYN (Position-Independent Executable file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x6d30
  Start of program headers:          64 (bytes into file)
  Start of section headers:          140328 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         13
  Size of section headers:           64 (bytes)
  Number of section headers:         31
  Section header string table index: 30
```
Entry point address:程序起点
Start of program headers:程序头表的偏移量

# 程序头表
面向系统,描述了各个"段",主要包含在运行时会用到的部分: 哪些需要加载的到内存,映射到哪里,权限如何
```fszz@fszz:~/ctf$ readelf -l /bin/sh

Elf file type is DYN (Position-Independent Executable file)
Entry point 0x4f10
There are 13 program headers, starting at offset 64

Program Headers:
  Type           Offset  文件中位置  VirtAddr  内存中位   PhysAddr
                 FileSiz            MemSiz              Flags  Align
  PHDR           0x0000000000000040 0x0000000000000040 0x0000000000000040
                 0x00000000000002d8 0x00000000000002d8  R      0x8
  INTERP         0x0000000000000318 0x0000000000000318 0x0000000000000318
                 0x000000000000001c 0x000000000000001c  R      0x1
      [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
  LOAD           0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x00000000000037d0 0x00000000000037d0  R      0x1000
  LOAD           0x0000000000004000 0x0000000000004000 0x0000000000004000
                 0x000000000001481d 0x000000000001481d  R E    0x1000
  LOAD           0x0000000000019000 0x0000000000019000 0x0000000000019000
                 0x0000000000004600 0x0000000000004600  R      0x1000
  LOAD           0x000000000001ddd0 0x000000000001edd0 0x000000000001edd0
                 0x0000000000001450 0x0000000000004100  RW     0x1000
  DYNAMIC        0x000000000001eb00 0x000000000001fb00 0x000000000001fb00
                 0x00000000000001f0 0x00000000000001f0  RW     0x8
  NOTE           0x0000000000000338 0x0000000000000338 0x0000000000000338
                 0x0000000000000030 0x0000000000000030  R      0x8
  NOTE           0x0000000000000368 0x0000000000000368 0x0000000000000368
                 0x0000000000000044 0x0000000000000044  R      0x4
  GNU_PROPERTY   0x0000000000000338 0x0000000000000338 0x0000000000000338
                 0x0000000000000030 0x0000000000000030  R      0x8
  GNU_EH_FRAME   0x000000000001afb4 0x000000000001afb4 0x000000000001afb4
                 0x000000000000060c 0x000000000000060c  R      0x4
  GNU_STACK      0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000000000 0x0000000000000000  RW     0x10
  GNU_RELRO      0x000000000001ddd0 0x000000000001edd0 0x000000000001edd0
                 0x0000000000001230 0x0000000000001230  R      0x1

 Section to Segment mapping:
  Segment Sections...
   00
   01     .interp
   02     .interp .note.gnu.property .note.gnu.build-id .note.ABI-tag .gnu.hash .dynsym .dynstr .gnu.version .gnu.version_r .rela.dyn .rela.plt
   03     .init .plt .plt.got .plt.sec .text .fini
   04     .rodata .eh_frame_hdr .eh_frame
   05     .init_array .fini_array .data.rel.ro .dynamic .got .data .bss
   06     .dynamic
   07     .note.gnu.property
   08     .note.gnu.build-id .note.ABI-tag
   09     .note.gnu.property
   10     .eh_frame_hdr
   11
   12     .init_array .fini_array .data.rel.ro .dynamic .got
```
load:表示要加载到内存的部分
dynamic:共享库

 Section to Segment mapping:描述了上面的每个段所包含的"节"
 节, 描述的是文件内部的更细致组织架构,在生成elf文件时,会将多个节按权限和加载属性合并,放到不同的段里面,节在运行时不是必须的
 用 `readelf -S`可以查看节头表