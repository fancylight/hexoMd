---
title: elf
date: 2018-09-04 08:34:55
tags:
- 原理
categories:
- C
---
# elf文件(Executable and Linkable Format)
[文档参考](http://www.skyfree.org/linux/references/ELF_Format.pdf)

-----
### 文件分类

- 可重定位文件:由编译器产生,具有重定位信息,没有程序头
- 可执行文件:由链接器产生
{%asset_img 1.png%}
上图说明了，一个目标文件生成可执行文件，然后加载到内存后的映射等，三个步骤。
    ELF头描述了生成该文件的系统的字的大小和字节序。ELF和节头部表之间每个部分都称为一个节（section）
    .text：已编译程序的机器代码
    .rodada：只读数据，比如printf语句中的格式串。
    .data：已经初始化的全局C变量。局部变量在运行时保存在栈中。即不再data节也不在bss节
    .bss：未初始化的全局C变量。不占据实际的空间，仅仅是一个占位符。所以未初始化变量不需要占据任何实际的磁盘空间。C++弱化BSS段。可能是没有，也可能有。
    .symtab：一个符号表，它存放“在程序中定义和引用的函数和全局变量的信息”。
    .rel.text：一个.text节中位置的列表。（将来重定位使用的）
    .rel.data：被模块引用或定义的任何全局变量的重定位信息。
    .debug：调试符号表，其内容是程序中定义的局部变量和类型定义。
    .line：原始C源程序的行号和.text节中机器指令之间的映射。
    .strtab：一个字符串表.
  
  程序头表通过段的形式将可执行文件分类，其描述了不同段具有的性质，比如说load，说明这部分内容是载入程序的，PT_GNU_RELRO，表示被定义为该段的内容在重定向后不能再被读取
  section table 则通过将文件本身的信息描述在不同的section中，如 text table表示了汇编代码。。
    
------
### 文件格式

{%blockquote%}
通过直接查看linux  `usr/include/elf.h`文件可以明确了解
{%endblockquote%}

{% asset_img elf格式.png elf两种典型格式 %}

- elf.h头文件

|名称|大小|用途|
|--|--|--|
|Elf32_Half|2|无符号整数|
|Elf64_Half|2|无符号整数|
 |Elf32_Word|4|无符号整数|
 |Elf32_Sword|4|带符号整数|
 |Elf64_Word|4|无符号整数|
 |Elf64_Sword|4|带符号整数|
 |Elf32_Xword|8|无符号整数|
 |Elf32_Sxword|8|带符号整数|
 |Elf64_Xword|8|带符号整数|
 |Elf64_Sxword|8|无符号整数|
|Elf32_Addr|4|地址|
|Elf64_Addr|8|地址|
|Elf32_Off|4|偏移|
|Elf64_Off|8|偏移|
| Elf32_Section|2|节|
 |Elf64_Section|2|节|
 | Elf32_Versym|2|符号信息|
 |Elf64_Versym|2|符号信息|
   
 -----
 ### efl头 elf header
- elf头:32位和64位分别占52和60字节
```
typedef struct
{
    unsigned char	e_ident[EI_NIDENT];	/* 魔数 */
    Elf32_Half	e_type;			/* 文件类型:可执行/可重入 */
    Elf32_Half	e_machine;		/* 机器 */
    Elf32_Word	e_version;		/* 文件版本
    Elf32_Addr	e_entry;		/* 入口点地址*/
    Elf32_Off	e_phoff;		/* 程序头距离文件起始偏移量*/
    Elf32_Off	e_shoff;		/* 节头表距离文件起始偏移量 */
    Elf32_Word	e_flags;		/* 处理器标志 */
    Elf32_Half	e_ehsize;		/*elf头大小 */
    Elf32_Half	e_phentsize;		/* 程序头中各条目大小 */
    Elf32_Half	e_phnum;		/* 程序头中条目数量 */
    Elf32_Half	e_shentsize;		/* 节头表中条目大小 */
    Elf32_Half	e_shnum;		/* 节头表中条目数量 */
    Elf32_Half	e_shstrndx;		/* 接头表在符号表中的下标 */
} Elf32_Ehdr;

typedef struct
{
    unsigned char	e_ident[EI_NIDENT];	/* Magic number and other info */
    Elf64_Half	e_type;			/* Object file type */
    Elf64_Half	e_machine;		/* Architecture */
    Elf64_Word	e_version;		/* Object file version */
    Elf64_Addr	e_entry;		/* Entry point virtual address */
    Elf64_Off	e_phoff;		/* Program header table file offset */
    Elf64_Off	e_shoff;		/* Section header table file offset */
    Elf64_Word	e_flags;		/* Processor-specific flags */
    Elf64_Half	e_ehsize;		/* ELF header size in bytes */
    Elf64_Half	e_phentsize;		/* Program header table entry size */
    Elf64_Half	e_phnum;		/* Program header table entry count */
    Elf64_Half	e_shentsize;		/* Section header table entry size */
    Elf64_Half	e_shnum;		/* Section header table entry count */
    Elf64_Half	e_shstrndx;		/* Section header string table index */
} Elf64_Ehdr;
```

 - 使用linux readelf命令可以读取elf文件,参考[elf命令](http://man.linuxde.net/readelf )  
 - 使用hexdump -x obj -n xx 按照16进制双字节排列 读取obj前xx字节数据
 - -x表示双排列 - n表示读取多少  -s 表示偏移
{%codeblock test.cpp lang:cpp %}
int sum(int *a,int n);
int array[2]={1,2};
int main()
{
    int val=sum(array,2);
    return val;
}
{%endcodeblock%}

{%codeblock sum.cpp lang:cpp %}
int sum(int *a,int n)
{
    int i,s=0;
    for(i=0;i<n;i++)
    {
        s+a[i];
    }
    return s;
}

{%endcodeblock%}
```
[root@riMlzL162793 link]# readelf -h test
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00 
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x4004c0
  Start of program headers:          64 (bytes into file)
  Start of section headers:          6648 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         9
  Size of section headers:           64 (bytes)
  Number of section headers:         30
  Section header string table index: 27
[root@riMlzL162793 link]# hexdump -x test -n 64
#此处每两个字节是按照高低位放置的
0000000    457f    464c    0102    0001    0000    0000    0000    0000 
0000010    0002    003e    0001    0000    04c0    0040    0000    0000 
0000020    0040    0000    0000    0000    19f8    0000    0000    0000
0000030    0000    0000    0040    0038    0009    0040    001e    001b
0000040
```
- 第一行:7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00 ,前四个字节7f454c46(0x45,0x4c,0x46是'e','l','f'对应的ascii编码）是一个魔数，表示这是一个ELF对象。02 表示elf64 01 表示小端补码,01,其他都为0
- 第二行:0002 表示为可执行文件,003e表示机器Advanced Micro Devices X86-64,0000 0001表示版本1,0000 0000 0040 04c0 表示 入口地址
- 第三行:0000 0000 0000 0040 表示progrom header table距离文件偏移(64), 0000 0000 0000 19f8表示section header table偏移(6648)
- 第四行:0000 0000 表示处理器标志,0040表示elf header大小(64),0038表示程序头表大小(56),0009表示progrom header entry数量(9) ,0040表示section header 大小 (64),001e section header entry(30),section header 字符串(string table)下标27
----
  
### 程序头表 progrom header
程序头表将可执行文件按照段进行分类

可执行文件或者共享目标文件的程序头部是一个结构数组，每个结构描述了一个段 或者系统准备程序执行所必需的其它信息。目标文件的“段”包含一个或者多个“节区”， 也就是“段内容(Segment Contents)”。程序头部仅对于可执行文件和共享目标文件 有意义。 可执行目标文件在 ELF 头部的 e_phentsize和e_phnum 成员中给出其自身程序头部 的大小。程序头部的数据结构:
```c++
typedef struct
{
    Elf32_Word	p_type;			/* Segment type */
    Elf32_Off	p_offset;		/* Segment file offset */
    Elf32_Addr	p_vaddr;		/* Segment virtual address */
    Elf32_Addr	p_paddr;		/* Segment physical address */
    Elf32_Word	p_filesz;		/* Segment size in file */
    Elf32_Word	p_memsz;		/* Segment size in memory */
    Elf32_Word	p_flags;		/* Segment flags */
    Elf32_Word	p_align;		/* Segment alignment */
} Elf32_Phdr;

typedef struct
{
    Elf64_Word	p_type;			/* Segment type */
    Elf64_Word	p_flags;		/* Segment flags */
    Elf64_Off	p_offset;		/* Segment file offset */
    Elf64_Addr	p_vaddr;		/* Segment virtual address */
    Elf64_Addr	p_paddr;		/* Segment physical address */
    Elf64_Xword	p_filesz;		/* Segment size in file */
    Elf64_Xword	p_memsz;		/* Segment size in memory */
    Elf64_Xword	p_align;		/* Segment alignment */
} Elf64_Phdr;
```
- type 此数组元素描述的段的类型，或者如何解释此数组元素的信息。具体如.
- offset 此成员给出从文件头到该段第一个字节的偏移。
- vaddr 此成员给出段的第一个字节将被放到内存中的虚拟地址。
- paddr 此成员仅用于与物理地址相关的系统中。
- filesz 此成员给出段在文件映像中所占的字节数。可以为 0。
- memsz 此成员给出段在内存映像中占用的字节数。可以为 0。
- flags 此成员给出与段相关的标志。
- align 可加载的进程段的 p_vaddr 和 p_offset 取值必须合适，相对于对页面大小的取模而言。此成员给出在文件中和内存中如何 对齐。数值 0 和 1 表示不需要对齐。否则 p_align 应该是个正整数，并且是 2 的幂次数，p_vaddr 和 p_offset 对 p_align 取模后应该相等。
  
{%codeblock 程序头宏定义%}
/* Legal values for p_type (segment type).  */
#define	PT_NULL		0		/* Program header table entry unused */
#define PT_LOAD		1		/* Loadable program segment */
#define PT_DYNAMIC	2		/* Dynamic linking information 动态链接信息*/
#define PT_INTERP	3		/* Program interpreter  程序解释器，linux下该段存放了动态链接器的位置*/
#define PT_NOTE		4		/* Auxiliary information  辅助信息*/
#define PT_SHLIB	5		/* Reserved 保留 */
#define PT_PHDR		6		/* Entry for header table itself 代表程序头本身*/
#define PT_TLS		7		/* Thread-local storage segment */
#define	PT_NUM		8		/* Number of defined types */
#define PT_LOOS		0x60000000	/* Start of OS-specific */
#define PT_GNU_EH_FRAME	0x6474e550	/* GCC .eh_frame_hdr segment */
#define PT_GNU_STACK	0x6474e551	/* Indicates stack executability */
#define PT_GNU_RELRO	0x6474e552	/* Read-only after relocation */
#define PT_LOSUNW	0x6ffffffa
#define PT_SUNWBSS	0x6ffffffa	/* Sun Specific segment */
#define PT_SUNWSTACK	0x6ffffffb	/* Stack segment */
#define PT_HISUNW	0x6fffffff
#define PT_HIOS		0x6fffffff	/* End of OS-specific */
#define PT_LOPROC	0x70000000	/* Start of processor-specific */
#define PT_HIPROC	0x7fffffff	/* End of processor-specific */
{%endcodeblock%}

{% asset_img efl_program_header.png%}
举例说明:
```
Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  PHDR           0x0000000000000040 0x0000000000400040 0x0000000000400040
                 0x00000000000001f8 0x00000000000001f8  R E    8
  INTERP         0x0000000000000238 0x0000000000400238 0x0000000000400238
                 0x000000000000001c 0x000000000000001c  R      1
      [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
  LOAD           0x0000000000000000 0x0000000000400000 0x0000000000400000
                 0x00000000000007e4 0x00000000000007e4  R E    200000
  LOAD           0x0000000000000de0 0x0000000000600de0 0x0000000000600de0
                 0x0000000000000254 0x0000000000000258  RW     200000
  DYNAMIC        0x0000000000000df8 0x0000000000600df8 0x0000000000600df8
                 0x0000000000000200 0x0000000000000200  RW     8
  NOTE           0x0000000000000254 0x0000000000400254 0x0000000000400254
                 0x0000000000000044 0x0000000000000044  R      4
  GNU_EH_FRAME   0x0000000000000690 0x0000000000400690 0x0000000000400690
                 0x000000000000003c 0x000000000000003c  R      4
  GNU_STACK      0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000000000 0x0000000000000000  RW     10
  GNU_RELRO      0x0000000000000de0 0x0000000000600de0 0x0000000000600de0
                 0x0000000000000220 0x0000000000000220  R      1

 Section to Segment mapping:
  Segment Sections...
   00     
   01     .interp 
   02     .interp .note.ABI-tag .note.gnu.build-id .gnu.hash .dynsym .dynstr .gnu.version .gnu.version_r .rela.dyn .rela.plt .init .plt .text .fini .rodata .eh_frame_hdr .eh_frame 
   03     .init_array .fini_array .jcr .dynamic .got .got.plt .data .bss 
   04     .dynamic 
   05     .note.ABI-tag .note.gnu.build-id 
   06     .eh_frame_hdr 
   07     
   08     .init_array .fini_array .jcr .dynamic .got 
[root@riMlzL162793 link]# hexdump -x test -n 120
0000000    457f    464c    0102    0001    0000    0000    0000    0000
0000010    0002    003e    0001    0000    04c0    0040    0000    0000
0000020    0040    0000    0000    0000    19f8    0000    0000    0000
0000030    0000    0000    0040    0038    0009    0040    001e    001b
0000040    0006    0000    0005    0000    0040    0000    0000    0000
0000050    0040    0040    0000    0000    0040    0040    0000    0000
0000060    01f8    0000    0000    0000    01f8    0000    0000    0000
0000070    0008    0000    0000    0000                                
0000078
```
由elf头可知program偏移为64字节处,因此0000040开始为program header，每个条目占用56字节
- 40行: 0000 0006 类型对应PT_PHDR ,0000 0005 表示RE(可读可执行),0000 0000 0000 0040表示偏移地址
- 50: 0000 0000 0040 0040 表示虚拟地址     0000 0000 0040 0040 表示物理地址
- 60: 0000 0000 0000 01f8 表示在可执行文件中的空间大小 ,01f8    0000    0000    0000表示实际运行时内存大小
- 70: 0000 0000 0000 0008 表示 对齐方式按照8字节对齐

 ### 节头表 section header table
 ```c++
 typedef struct
{
    Elf32_Word	sh_name;		/* Section name (string tbl index) 节名 */
    Elf32_Word	sh_type;		/* Section type 类型*/
    Elf32_Word	sh_flags;		/* Section flags 标志*/
    Elf32_Addr	sh_addr;		/* Section virtual addr at execution 执行虚拟地址*/
    Elf32_Off	sh_offset;		/* Section file offset 文件偏移*/
    Elf32_Word	sh_size;		/* Section size in bytes 大小*/
    Elf32_Word	sh_link;		/* Link to another section */
    Elf32_Word	sh_info;		/* Additional section information */
    Elf32_Word	sh_addralign;		/* Section alignment 对齐方式*/
    Elf32_Word	sh_entsize;		/* Entry size if section holds table */
} Elf32_Shdr;

typedef struct
{
    Elf64_Word	sh_name;		/* Section name (string tbl index) */
    Elf64ord	sh_type;		/* Section type */
    Elf64_Xword	sh_flags;		/* Section flags */
    Elf64_Addr	sh_addr;		/* Section virtual addr at execution */
    Elf64_Off	sh_offset;		/* Section file offset */
    Elf64_Xword	sh_size;		/* Section size in bytes */
    Elf64_Word	sh_link;		/* Link to another section */
    Elf64_Word	sh_info;		/* Additional section information */
    Elf64_Xword	sh_addralign;		/* Section alignment */
    Elf64_Xword	sh_entsize;		/* Entry size if section holds table */
} Elf64_Shdr;
 ```
 这个结构体中sh_type 对应了不同种类的section
 ```
 /* Legal values for sh_type (section type).  */

#define SHT_NULL	  0		/* Section header table entry unused */
#define SHT_PROGBITS	  1		/* Program data */
#define SHT_SYMTAB	  2		/* Symbol table */
#define SHT_STRTAB	  3		/* String table */
#define SHT_RELA	  4		/* Relocation entries with addends */
#define SHT_HASH	  5		/* Symbol hash table */
#define SHT_DYNAMIC	  6		/* Dynamic linking information */
#define SHT_NOTE	  7		/* Notes */
#define SHT_NOBITS	  8		/* Program space with no data (bss) */
#define SHT_REL		  9		/* Relocation entries, no addends */
#define SHT_SHLIB	  10		/* Reserved */
#define SHT_DYNSYM	  11		/* Dynamic linker symbol table */
#define SHT_INIT_ARRAY	  14		/* Array of constructors */
#define SHT_FINI_ARRAY	  15		/* Array of destructors */
#define SHT_PREINIT_ARRAY 16		/* Array of pre-constructors */
#define SHT_GROUP	  17		/* Section group */
#define SHT_SYMTAB_SHNDX  18		/* Extended section indeces */
#define	SHT_NUM		  19		/* Number of defined types.  */
#define SHT_LOOS	  0x60000000	/* Start OS-specific.  */
#define SHT_GNU_ATTRIBUTES 0x6ffffff5	/* Object attributes.  */
#define SHT_GNU_HASH	  0x6ffffff6	/* GNU-style hash table.  */
#define SHT_GNU_LIBLIST	  0x6ffffff7	/* Prelink library list */
#define SHT_CHECKSUM	  0x6ffffff8	/* Checksum for DSO content.  */
#define SHT_LOSUNW	  0x6ffffffa	/* Sun-specific low bound.  */
#define SHT_SUNW_move	  0x6ffffffa
#define SHT_SUNW_COMDAT   0x6ffffffb
#define SHT_SUNW_syminfo  0x6ffffffc
#define SHT_GNU_verdef	  0x6ffffffd	/* Version definition section.  */
#define SHT_GNU_verneed	  0x6ffffffe	/* Version needs section.  */
#define SHT_GNU_versym	  0x6fffffff	/* Version symbol table.  */
#define SHT_HISUNW	  0x6fffffff	/* Sun-specific high bound.  */
#define SHT_HIOS	  0x6fffffff	/* End OS-specific type */
#define SHT_LOPROC	  0x70000000	/* Start of processor-specific */
#define SHT_HIPROC	  0x7fffffff	/* End of processor-specific */
#define SHT_LOUSER	  0x80000000	/* Start of application-specific */
#define SHT_HIUSER	  0x8fffffff	/* End of application-specific */
 ```
 section table
```
[root@riMlzL162793 link]# readelf -S test
There are 30 section headers, starting at offset 0x19f8:

Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .interp           PROGBITS         0000000000400238  00000238
       000000000000001c  0000000000000000   A       0     0     1
  [ 2] .note.ABI-tag     NOTE             0000000000400254  00000254
       0000000000000020  0000000000000000   A       0     0     4
  [ 3] .note.gnu.build-i NOTE             0000000000400274  00000274
       0000000000000024  0000000000000000   A       0     0     4
  [ 4] .gnu.hash         GNU_HASH         0000000000400298  00000298
       000000000000001c  0000000000000000   A       5     0     8
  [ 5] .dynsym           DYNSYM           00000000004002b8  000002b8
       0000000000000090  0000000000000018   A       6     1     8
  [ 6] .dynstr           STRTAB           0000000000400348  00000348
       00000000000000a9  0000000000000000   A       0     0     1
  [ 7] .gnu.version      VERSYM           00000000004003f2  000003f2
       000000000000000c  0000000000000002   A       5     0     2
  [ 8] .gnu.version_r    VERNEED          0000000000400400  00000400
       0000000000000020  0000000000000000   A       6     1     8
  [ 9] .rela.dyn         RELA             0000000000400420  00000420
       0000000000000018  0000000000000018   A       5     0     8
  [10] .rela.plt         RELA             0000000000400438  00000438
       0000000000000030  0000000000000018  AI       5    12     8
  [11] .init             PROGBITS         0000000000400468  00000468
       000000000000001a  0000000000000000  AX       0     0     4
  [12] .plt              PROGBITS         0000000000400490  00000490
       0000000000000030  0000000000000010  AX       0     0     16
  [13] .text             PROGBITS         00000000004004c0  000004c0
       00000000000001b2  0000000000000000  AX       0     0     16
  [14] .fini             PROGBITS         0000000000400674  00000674
       0000000000000009  0000000000000000  AX       0     0     4
  [15] .rodata           PROGBITS         0000000000400680  00000680
       0000000000000010  0000000000000000   A       0     0     8
  [16] .eh_frame_hdr     PROGBITS         0000000000400690  00000690
       000000000000003c  0000000000000000   A       0     0     4
  [17] .eh_frame         PROGBITS         00000000004006d0  000006d0
       0000000000000114  0000000000000000   A       0     0     8
  [18] .init_array       INIT_ARRAY       0000000000600de0  00000de0
       0000000000000008  0000000000000000  WA       0     0     8
  [19] .fini_array       FINI_ARRAY       0000000000600de8  00000de8
       0000000000000008  0000000000000000  WA       0     0     8
  [20] .jcr              PROGBITS         0000000000600df0  00000df0
       0000000000000008  0000000000000000  WA       0     0     8
  [21] .dynamic          DYNAMIC          0000000000600df8  00000df8
       0000000000000200  0000000000000010  WA       6     0     8
  [22] .got              PROGBITS         0000000000600ff8  00000ff8
       0000000000000008  0000000000000008  WA       0     0     8
  [23] .got.plt          PROGBITS         0000000000601000  00001000
       0000000000000028  0000000000000008  WA       0     0     8
  [24] .data             PROGBITS         0000000000601028  00001028
       000000000000000c  0000000000000000  WA       0     0     4
  [25] .bss              NOBITS           0000000000601034  00001034
       0000000000000004  0000000000000000  WA       0     0     1
  [26] .comment          PROGBITS         0000000000000000  00001034
       000000000000002d  0000000000000001  MS       0     0     1
  [27] .shstrtab         STRTAB           0000000000000000  00001061
       0000000000000108  0000000000000000           0     0     1
  [28] .symtab           SYMTAB           0000000000000000  00001170
       0000000000000648  0000000000000018          29    46     8
  [29] .strtab           STRTAB           0000000000000000  000017b8
       000000000000023e  0000000000000000    
```
hexdump
 ```
0001a38    001b    0000    0001    0000    0002    0000    0000    0000
0001a48    0238    0040    0000    0000    0238    0000    0000    0000
0001a58    001c    0000    0000    0000    0000    0000    0000    0000
0001a68    0001    0000    0000    0000    0000    0000    0000    0000
 ```
 正确显示了对应第2个section  interp
 ### 符号表条目 symble table entry
 数据结构
 ```
 
typedef struct
{
  Elf32_Word	st_name;		/* Symbol name (string tbl index) */
  Elf32_Addr	st_value;		/* Symbol value */
  Elf32_Word	st_size;		/* Symbol size */
  unsigned char	st_info;		/* Symbol type and binding */
  unsigned char	st_other;		/* Symbol visibility */
  Elf32_Section	st_shndx;		/* Section index */
} Elf32_Sym;

typedef struct
{
  Elf64_Word	st_name;		/* Symbol name (string tbl index) */
  unsigned char	st_info;		/* Symbol type and binding */
  unsigned char st_other;		/* Symbol visibility */
  Elf64_Section	st_shndx;		/* Section index */
  Elf64_Addr	st_value;		/* Symbol value */
  Elf64_Xword	st_size;		/* Symbol size */
} Elf64_Sym;
 ```
 {%asset_img 符号表解释.png%}
 - 其中st_info st_other都是分为4个bit代表一个信息
 linux下：
 ```
 [root@riMlzL162793 link]# readelf -s test.o

Symbol table '.symtab' contains 11 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
     1: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS test.cpp
     2: 0000000000000000     0 SECTION LOCAL  DEFAULT    1 
     3: 0000000000000000     0 SECTION LOCAL  DEFAULT    3 
     4: 0000000000000000     0 SECTION LOCAL  DEFAULT    4 
     5: 0000000000000000     0 SECTION LOCAL  DEFAULT    6 
     6: 0000000000000000     0 SECTION LOCAL  DEFAULT    7 
     7: 0000000000000000     0 SECTION LOCAL  DEFAULT    5 
     8: 0000000000000000     8 OBJECT  GLOBAL DEFAULT    3 array
     9: 0000000000000000    31 FUNC    GLOBAL DEFAULT    1 main
    10: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND _Z3sumPii

 ```
 - 最后三项分别就是 array全局符号,main 全局符号,UND_Z3sumPii就是sum这个符号,它是未定义的所以type是notype,并且没有在该obj中任意一节中.
 ### 可重定位符号条目 Relocation table entry
 通过表示在符号表中的位置，以及在文件中的偏移量代表重定位信息
 ```
 /* Relocation table entry without addend (in section of type SHT_REL).  */

typedef struct
{
  Elf32_Addr	r_offset;		/* Address */
  Elf32_Word	r_info;			/* Relocation type and symbol index */
} Elf32_Rel;

/* I have seen two different definitions of the Elf64_Rel and
   Elf64_Rela structures, so we'll leave them out until Novell (or
   whoever) gets their act together.  */
/* The following, at least, is used on Sparc v9, MIPS, and Alpha.  */

typedef struct
{
  Elf64_Addr	r_offset;		/* Address */
  Elf64_Xword	r_info;			/* Relocation type and symbol index */
} Elf64_Rel;

/* Relocation table entry with addend (in section of type SHT_RELA).  */

typedef struct
{
  Elf32_Addr	r_offset;		/* Address */
  Elf32_Word	r_info;			/* Relocation type and symbol index */
  Elf32_Sword	r_addend;		/* Addend */
} Elf32_Rela;
 ```