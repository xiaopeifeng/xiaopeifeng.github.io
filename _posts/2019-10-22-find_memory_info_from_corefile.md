---
layout: default
title: find detail memory info from coredump file
---

Recently I write a C project at work, and this project has a log module depended on a tiny third party source file. I wrote a logger callback implementation for the log interface.  the  callback function will format the log content and write it to log file.  I use the synchrous writing log pattern, because it's not a performance sensitive program. So it will take up worker thread's time, and the buffer will be flush to disk every N seconds. Many programs usually use another asynchorous pattern which has a special log thread to write log, this pattern will not wast worker thread's time, but have you think that both of these two patterns has a same problem while the program crashed? log writting module usually won't write its content to  disk directly for performace consideration. So if coredump happened, some log content will be lost which is very valuable for analysing the bug.

​	core file is valuable for analysing the bug, if we want to see the last log content before core dump event happened?

gdb is a very powerful tool, and it can do that for us. now I give you an example code:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

typedef void (*function_t)(void);

typedef struct Foo {
  function_t func;
  char str[8];
} Foo;

static
void tag_func() {}

int main(int argc, char** argv) {
  Foo *foo_list = malloc(8*sizeof(Foo));
  memset(foo_list, 0, 8*sizeof(Foo));

  int i;
  for (i = 0; i < 8; ++i) {
    foo_list[i].func = tag_func;
    foo_list[i].str[0] = '0' + i;
    foo_list[i].str[1] = '0' + i;
    foo_list[i].str[2] = '0' + i;
  }

  abort();
}
```

the code will crash at last line, but if you want to see the *str* in all the *foo_list*, how can we get it?

firstly let's launch gdb with coredump file

```shell
[fxp@centos test]$ gdb ./a.out core.1967
```

*foo_list*'s address is:

```shell
(gdb) bt
#0  0x00007f945ef1d2c7 in __GI_raise (sig=sig@entry=6) at ../nptl/sysdeps/unix/sysv/linux/raise.c:55
#1  0x00007f945ef1e9b8 in __GI_abort () at abort.c:90
#2  0x000000000040066c in main (argc=1, argv=0x7ffd8d5306e8) at coredump_mem_find.c:27
(gdb) frame 2
#2  0x000000000040066c in main (argc=1, argv=0x7ffd8d5306e8) at coredump_mem_find.c:27
27	  abort();
(gdb) p foo_list 
$2 = (Foo *) 0x18ea010
```

now it's key point part, let's get the *tag_func* address,

```shell
(gdb) info address tag_func 
Symbol "tag_func" is a function at address 0x4005ad.
```

then, we get the specified heap block where foo_list located,

```shell
(gdb) info files 
Symbols from "/home/fxp/test/a.out".
Local core dump file:
	`/home/fxp/test/core.2481', file type elf64-x86-64.
	0x0000000000400000 - 0x0000000000401000 is load1
	0x0000000000600000 - 0x0000000000601000 is load2
	0x0000000000601000 - 0x0000000000602000 is load3
	0x00000000018ea000 - 0x000000000190b000 is load4
	0x00007f945eee7000 - 0x00007f945eee8000 is load5a
	0x00007f945eee8000 - 0x00007f945eee8000 is load5b
	0x00007f945f0a9000 - 0x00007f945f0a9000 is load6
	0x00007f945f2a9000 - 0x00007f945f2ad000 is load7
	0x00007f945f2ad000 - 0x00007f945f2af000 is load8
	0x00007f945f2af000 - 0x00007f945f2b4000 is load9
	0x00007f945f2b4000 - 0x00007f945f2b5000 is load10a
	0x00007f945f2b5000 - 0x00007f945f2b5000 is load10b
	0x00007f945f4c9000 - 0x00007f945f4cc000 is load11
	0x00007f945f4d4000 - 0x00007f945f4d5000 is load12
	0x00007f945f4d5000 - 0x00007f945f4d6000 is load13
	0x00007f945f4d6000 - 0x00007f945f4d7000 is load14
	0x00007f945f4d7000 - 0x00007f945f4d8000 is load15
	0x00007ffd8d511000 - 0x00007ffd8d532000 is load16
	0x00007ffd8d557000 - 0x00007ffd8d559000 is load17
	0xffffffffff600000 - 0xffffffffff601000 is load18
```

you can see address 0x18ea010 is in the region load4, so we can find all memory block will contain value *0x4005ad*

```shell
(gdb) find 0x00000000018ea000,0x000000000190b000,0x4005ad
0x18ea010
0x18ea020
0x18ea030
0x18ea040
0x18ea050
0x18ea060
0x18ea070
0x18ea080
warning: Unable to access 7037 bytes of target memory at 0x1909484, halting search.
8 patterns found.
```

as you can see, total 8 patterns was matched which is coherent to our code. next we can make a test if the memory block contain the right string content. as you konow, pointer take 8 bytes length, so we can get the address's content by directives

```shell
(gdb) x /s 0x18ea010+8
0x18ea018:	"000"
(gdb) x /s 0x18ea020+8
0x18ea028:	"111"
(gdb) x /s 0x18ea030+8
0x18ea038:	"222"
(gdb) x /s 0x18ea040+8
0x18ea048:	"333"
(gdb) x /s 0x18ea050+8
0x18ea058:	"444"
(gdb) x /s 0x18ea060+8
0x18ea068:	"555"
(gdb) x /s 0x18ea070+8
0x18ea078:	"666"
(gdb) x /s 0x18ea080+8
0x18ea088:	"777"
```

the key point is the *tag_func*, it's name is a hint, we make a fix tag for every target memroy block, so we can find the memory content mapped from the corefile with the help of gdb memory find directive.

