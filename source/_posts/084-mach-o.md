---
title: 『底层探索』10 - Mach-O 文件分析
toc: true
donate: false
tags: []
date: 2020-09-28 19:57:26
categories: [底层探索]
---

不同的系统下，可执行文件的格式也是不同的。我们今天探索的是 iOS 的 Mach-O 文件格式。

<!-- more -->

## 前言

我们编写的程序经过编译后会生成一个可执行文件，如果我们想要运行这个可执行文件，也就是运行我们编写的程序，离不开操作系统的支持。

那么对于操作系统来说，肯定要知道可执行文件的格式。可执行文件格式跟操作系统和编译器密切相关，不同的系统平台下有不同的格式。

Windows 下是 PE（Portable Executable）格式，Linux 下是 ELF（Executable Linkable Format）格式。iOS 和 Mac OS 是 Mach-O 格式的。本文即将探索的是 `Mach-O` 。

在 Mac OS 系统中，我们可以通过 file 来查看文件格式。

```shell
file AppLoad
AppLoad: Mach-O 64-bit executable arm64
```

## Mach-O 简介

Mach-O 是 Mach Object 文件格式的缩写。它是一种用于可执行文件、目标代码、动态库的文件格式。作为 `a.out` 格式的替代，Mach-O 提供了更强的扩展性。并提升了符号表中信息的访问速度。

可执行格式决定了二进制文件中的代码和数据读入内存中的顺序，代码和数据的顺序会影响内存的使用和分页活动，因此会直接影响程序的性能。

一个 Mach-O 二进制文件是以 Segment（段） 的方式组织的。每个 Segment 包含 1 个或多个 Section (节)。Segment 在内存中是从 page boundary（页边界）开始的，Section 不是页面对齐的。

可能有同学不知道内存分页，这里简单说下。内存分页是一种内存管理技术，用于控制如何共享计算机或虚拟机的内存资源。操作系统会将虚拟内存和物理内存都划分为多个同样大小（4KB）的单元，每个单元称为页。

Segment 的大小由它包含的所有 Section 加起来的字节数，然后向上取整到下一个页边界。所有 Segment 的大小始终是 4KB 的整数倍。

Mach-O 中的 Segment 和 Section 是根据用途进行命名的。

- Segment 命名是 `__` 后跟全大写字母，如 __TEXT。
- Section 命名是 `__` 后跟全小写字母，如 __text。

## Mach-O 文件结构

Mach-O 文件主要由 3 部分组成（如图所示）

![](https://raw.githubusercontent.com/muhlenxi/blog-images/master/img/mach_o_segments.png)

- Header，用于标识二进制文件为 Mach-O 可执行文件，除此之外还包含二进制文件的基本信息，包括
 -  指令集架构、CPU 类型和详细类型、文件类型、加载命令的数量、大小及标志位等。
- Load Commands, 这部分内容紧跟在 header 后面，是用于描述如何加载后面 Data 的数据，包括
    - 内存布局信息、链接信息、符号表的位置、共享库的名称等
- Data，是二进制文件中的数据部分，是以 Section 的方式组织的，是特定类型的代码或数据。
    - 程序源代码编译后的机器指令一般放在代码段，也就是 `__TEXT`，这个 Segment 是只读的。
    - 全局变量和局部静态变量数据一般放在数据段，也就是 `__DATA`，这个 Segment 可读可写。

## 用 MachOView 查看可执行文件

下图是我们通过 `MachOView` 打开一个 App 的可执行文件得到的结构图。

![](https://raw.githubusercontent.com/muhlenxi/blog-images/master/img/macho.jpg)

__TEXT 中的主要 section 如下：


| Section | Description |
|---------|-------------|
| __text    |   The compiled machine code for the executable        |
| __const    |   The general constant data for the executable        |
| __cstring |   Literal string constants (quoted strings in source  code)       |
| __picsymbol_stub   |   Position-independent code stub routines used by the dynamic linker (dyld).       |


__DATA 中的主要 section 如下


| Section | Description |
|---------|-------------|
|    __data     |    Initialized global variables (for example int a = 1; or static int a = 1;).  |
| __const | Constant data needing relocation (for example, char * const p = "foo";). |
| __bss | Uninitialized static variables (for example, static int a;). |
| __common | Uninitialized external globals (for example, int a; outside function blocks). |
| __dyld | A placeholder section, used by the dynamic linker. |
| __la_symbol_ptr | “Lazy” symbol pointers. Symbol pointers for each undefined function called by the executable.|
| __nl_symbol_ptr |“Non lazy” symbol pointers. Symbol pointers for each undefined data symbol referenced by the executable. |

## 可执行文件中的代码结构表示

### mach_header_64

在 `usr/include/mach-o/loader.h` 文件中，我们可以找到 `mach_header` 和 `mach_header_64` 的定义。64位架构比 32 位架构少了一个 `reserved` 字段。

```c
struct mach_header_64 {
	uint32_t	magic;		/* mach magic number identifier */
	cpu_type_t	cputype;	/* cpu specifier */
	cpu_subtype_t	cpusubtype;	/* machine specifier */
	uint32_t	filetype;	/* type of file */
	uint32_t	ncmds;		/* number of load commands */
	uint32_t	sizeofcmds;	/* the size of all the load commands */
	uint32_t	flags;		/* flags */
	uint32_t	reserved;	/* reserved */
};
```

我们可以通过 magic 的值来快速识别是哪个架构，使用的是大端还是小端。

```
#define	MH_MAGIC	0xfeedface	 /* the mach magic number */
#define MH_CIGAM	0xcefaedfe	/* NXSwapInt(MH_MAGIC) */

#define MH_MAGIC_64 0xfeedfacf  /* the 64-bit mach magic number */
#define MH_CIGAM_64 0xcffaedfe /* NXSwapInt(MH_MAGIC_64) */
```

filetype 用于表示 mach-o 文件的类型。我们看看定义了哪些类型, 我摘录了部分类型，完整的请查阅 `loader.h` 文件。

```c
#define	MH_OBJECT	0x1		/* relocatable object file */
#define	MH_EXECUTE	0x2		/* demand paged executable file */
#define	MH_FVMLIB	0x3		/* fixed VM shared library file */
#define	MH_CORE		0x4		/* core file */
#define	MH_PRELOAD	0x5		/* preloaded executable file */
#define	MH_DYLIB	0x6		/* dynamically bound shared library */
#define	MH_DYLINKER	0x7		/* dynamic link editor */
#define	MH_BUNDLE	0x8		/* dynamically bound bundle file */
#define	MH_DYLIB_STUB	0x9		/* shared library stub for static
					   linking only, no section contents */
#define	MH_DSYM		0xa		/* companion file with only debug
					   sections */
```

### load_command

每个 load command 有两个属性，一个是 type，一个是 total size。

```c
struct load_command {
	uint32_t cmd;		/* type of load command */
	uint32_t cmdsize;	/* total size of command in bytes */
};
```

根据 load command 的 define，可以得到如下的类型，我摘录了部分类型，完整的请查阅 `loader.h` 文件。

```c
#define LC_REQ_DYLD 0x80000000

/* Constants for the cmd field of all load commands, the type */
#define	LC_SEGMENT	0x1	/* segment of this file to be mapped */
#define	LC_SYMTAB	0x2	/* link-edit stab symbol table info */
#define	LC_SYMSEG	0x3	/* link-edit gdb symbol table info (obsolete) */
#define	LC_THREAD	0x4	/* thread */
#define	LC_UNIXTHREAD	0x5	/* unix thread (includes a stack) */
#define	LC_LOADFVMLIB	0x6	/* load a specified fixed VM shared library */
#define	LC_IDFVMLIB	0x7	/* fixed VM shared library identification */
#define	LC_IDENT	0x8	/* object identification info (obsolete) */
#define LC_FVMFILE	0x9	/* fixed VM file inclusion (internal use) */
#define LC_PREPAGE      0xa     /* prepage command (internal use) */
#define	LC_DYSYMTAB	0xb	/* dynamic link-edit symbol table info */
#define	LC_LOAD_DYLIB	0xc	/* load a dynamically linked shared library */
#define	LC_ID_DYLIB	0xd	/* dynamically linked shared lib ident */
#define LC_LOAD_DYLINKER 0xe	/* load a dynamic linker */
#define LC_ID_DYLINKER	0xf	/* dynamic linker identification */
```

我们再看看 LC_SEGMENT 类型 Load Command 的结构体表示。

```
struct segment_command_64 { /* for 64-bit architectures */
	uint32_t	cmd;		/* LC_SEGMENT_64 */
	uint32_t	cmdsize;	/* includes sizeof section_64 structs */
	char		segname[16];	/* segment name */
	uint64_t	vmaddr;		/* memory address of this segment */
	uint64_t	vmsize;		/* memory size of this segment */
	uint64_t	fileoff;	/* file offset of this segment */
	uint64_t	filesize;	/* amount to map from the file */
	vm_prot_t	maxprot;	/* maximum VM protection */
	vm_prot_t	initprot;	/* initial VM protection */
	uint32_t	nsects;		/* number of sections in segment */
	uint32_t	flags;		/* flags */
};
```

### Section

对于每个 Section，是这样表示的，下面是 64 架构的 Section。

```c
struct section_64 { /* for 64-bit architectures */
	char		sectname[16];	/* name of this section */
	char		segname[16];	/* segment this section goes in */
	uint64_t	addr;		/* memory address of this section */
	uint64_t	size;		/* size in bytes of this section */
	uint32_t	offset;		/* file offset of this section */
	uint32_t	align;		/* section alignment (power of 2) */
	uint32_t	reloff;		/* file offset of relocation entries */
	uint32_t	nreloc;		/* number of relocation entries */
	uint32_t	flags;		/* flags (section type and attributes)*/
	uint32_t	reserved1;	/* reserved (for offset or index) */
	uint32_t	reserved2;	/* reserved (for count or sizeof) */
	uint32_t	reserved3;	/* reserved */
};
```

## 参考资料

- [1、Overview of the Mach-O Executable Format](https://developer.apple.com/library/archive/documentation/Performance/Conceptual/CodeFootprint/Articles/MachOOverview.html#//apple_ref/doc/uid/20001860-BAJGJEJC)
- [2、Mach-O File Format Reference](http://mirror.informatimago.com/next/developer.apple.com/documentation/DeveloperTools/Conceptual/MachORuntime/8rt_file_format/chapter_10_section_1.html)

## 后记

我是穆哥，卖码维生的一朵浪花。我们下回见。 