https://blog.csdn.net/robertsong2004/article/details/38499995


这是一个日本人写的用户态下的函数tracer, 我们知道系统调用可以用strace, 库调用可以使用ltrace, 但是linux下竟然没有一个比较有名的用户程序的tracer, 这真是比较奇怪。

这个工具好的地方就是用ptrace系统调用来实现，只要跟踪的程序没有被strip，就可以使用，而不要重新编译程序。而另一种函数跟踪的方式（使用gcc -finstruction-functions），目标程序必须要重新编译，这个就大大降低了tracer的实用性。

但是目前这个工具只支持x86架构，arm之类的嵌入式环境不支持。
目前还剩下实现机制这一部分没有翻译完。

原文来自： http://binary.nahi.to/hogetrace/

tracef - function call tracer
该网页尚未完成。
之前用的名字hogetrace比较过分，自重起见改成tracef。
TOC
概要
执行用例
执行环境 (OS)
下载
编译环境
编译方法
可解析的程序
例子
fork程序的解析
exec程序的解析
递归
多线程
大家喜闻乐见的抛出异常
main之前调用的函数
限制事项
命令行选项
实现机制
类似的工具
概要
tracef是、面向Linux的「函数调用追踪器」。 和一般在Linux发行版使用的ltrace相类似、但是其有下面的特征和不同点。

不单是调用DSO(DLL)里的库函数、自己的函数的调用也可以追踪
函数调用的父子关系用可视化的树状图表示
可以表示实现函数的文件名和行号
根据上述的这些特征、

想了解未知的大型程序的执行的时候
阅读源代码时想得到一些额外的信息的时候
(特别是C++程序中) 想简单确认main函数前后做了什么样的初始化操作的时候
...
等等，都可以灵活利用该工具。但是遗憾的是，函数调用时参数的信息没有ltrace那么详细，用于调试还比较困难。目前我手头上的用C++写的比较大的执行文件(.text的大小约为5MB、text/weak的symbol数量是2万左右、进程/线程数量有几十个)的解析都没有什么问题。

如果解析对象没有被strip，就不需要再编译。只要是能使用gdb来调试的可执行文件、也能用tracef来跟踪。除了作为解析対象的可执行文件所包含的「自身函数」的调用以外、如ltrace表示的内容那样、库函数的调用状况也在某种程度上可以表示（命令行选项可以选项是否表示）。

tracef和、ltrace/strace、或gdb一样、通过使用Linux kernel的ptrace(2)系统调用、从别的进程来观察作为解析対象的程序，并显观察状况。

执行用例
例如要追踪下面的程序的话、

#include <stdio.h>

void my_func_2() {
  puts("hello, world!");
}

void my_func_1() {
  my_func_2();
}

int main(int argc, char** argv) {
  my_func_1();
  fflush(stdout);
  return 0;
}
結果如下所示。

$ gcc -g -o hello hello.c 
$ tracef --plt --line-numbers --call-tree hello

[pid 30126] +++ process 30126 attached (ppid 30125) +++
[pid 30126] === symbols loaded: './hello' ===
[pid 30126] ==> _start() at 0x08048300
[pid 30126]    ==> __libc_start_main@plt() at 0x080482cc
[pid 30126]       ==> __libc_csu_init() at 0x08048440
[pid 30126]          ==> _init() at 0x08048294
[pid 30126]             ==> call_gmon_start() at 0x08048324
[pid 30126]             <== call_gmon_start() [eax = 0x0]
[pid 30126]             ==> frame_dummy() at 0x080483b0
[pid 30126]             <== frame_dummy() [eax = 0x0]
[pid 30126]             ==> __do_global_ctors_aux() at 0x080484b0
[pid 30126]             <== __do_global_ctors_aux() [eax = 0xffffffff]
[pid 30126]          <== _init() [eax = 0xffffffff]
[pid 30126]       <== __libc_csu_init() [eax = 0x8049514]
[pid 30126]       ==> main() at 0x080483f5 [/home/sato/tracef/sample/hello.c:14] 
[pid 30126]          ==> my_func_1() at 0x080483e8 [/home/sato/tracef/sample/hello.c:9] 
[pid 30126]             ==> my_func_2() at 0x080483d4 [/home/sato/tracef/sample/hello.c:4] 
[pid 30126]                ==> puts@plt() at 0x080482ec
[pid 30126]                <== puts@plt() [eax = 0xe]
[pid 30126]             <== my_func_2() [eax = 0xe]
[pid 30126]          <== my_func_1() [eax = 0xe]
[pid 30126]          ==> fflush@plt() at 0x080482dc
hello, world!
[pid 30126]          <== fflush@plt() [eax = 0x0]
[pid 30126]       <== main() [eax = 0x0]
[pid 30126]       ==> _fini() at 0x080484d8
[pid 30126]          ==> __do_global_dtors_aux() at 0x08048350
[pid 30126]          <== __do_global_dtors_aux() [eax = 0x0]
[pid 30126]       <== _fini() [eax = 0x0]
[pid 30126] +++ process 30126 detached (ppid 30125) +++
tracef: done
main()调用my_func_1()、my_func_1()调用my_func_2()的过程被显示。另外、 定义这些函数的源文件名也会显示。

这次因为追加了--synthetic选项、puts@plt() 等、DSO里的函数调用也能追踪到。不使用--synthetic 选项的话、仅仅是单纯的追踪程序自身的函数。根据不同的选项，函数的参数的简单的信息也能表示。

执行环境（ＯＳ）
Linux
kernel 2.6以后的版本
x86, x86_64(一部分的功能被限制)
下载
tracef-0.16.tar.gz (支持rpmbuild -t, GPLv3)
tracef-0.16-1.src.rpm
tracef-0.16-1.i386.rpm
编译环境
Fedora Core 6 / Fedora 7
gcc-c++
libstdc++-devel
binutils-devel
elfutils-libelf-devel
boost-devel
RHEL4 / CentOS4
gcc-c++
libstdc++-devel
binutils
elfutils-libelf-devel
boost-devel
Ubuntu 7.04
g++
binutils-dev
elfutils
libelf-dev
boost*
libdwarf
需要额外的libdwarf包。

libdwarf-20070703-3.src.rpm
libdwarf-20070703-3.i386.rpm (Ubuntu 7.04/x86 可用alien -i导入)
编译方法
$ tar xvzf tracef-0.1.tar.gz 
$ cd tracef-0.1
$ ./configure
$ cd src
$ make
make结果生成的 src/tracef 就是追踪器。这个二进制文件(tracef)、不安装在/usr/local/bin 下也可以、放在自己想放的地方、单独执行也没有问题。这里准备了一些测试用例 sample/用于测试结果。

$ cd ../sample
$ ./sample.sh
执行sample.sh后、会编译并链接若干程序，这些程序被tracef解析的结果会放在sample/logs/下。 

可解析的程序
是否需要重新编译
解析对象没有被strip的话就不需要重新编译。gdb能解析的二进制就不需要重新编译。

另外、解析対象是有用 -g 编译的话、输出的信息就会增多。例如行号的信息、参数信息都能输出。解析対象は、最好是用-O0 编译的、但这不是必须的。被优化的场合下、一部分的函数调用可能无法被检测出来。

被strip后的二进制的话，就算被追踪了tracef也不会异常退出，但不会输出任何解析結果。

详细
是否被strip、可以使用file命令来确认输出 "stripped" "not stipped"的哪一个。或是用 readelf -S 命令来判断二进制文件是否存在.symtab 段。 下面的例子的话a.out就没有被strip。

$ file a.out
a.out: ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.9, not stripped
$ readelf -S a.out | grep '\.symtab'
  [40] .symtab           SYMTAB          00000000 5d20b4 00d4d0 10     41 1056  4
另外、因为/lib/lib*.a 或 /usr/lib/lib*.a 文件 包含.symtab、如果链接这些 lib*.a 文件的话，这些文件里包含的函数也变成追踪的対象。不希望追踪这些函数的话，请不要使用.a 文件而是链接到 .so上。

程序用 gcc -g编译时就会包含调试信息、追踪结果里就能够包含定义该函数的源文件名、行编号。另外，一部分函数的参数信息也能表示。。程序是否是用-g (或 -ggdb 或 -g3 等等)编译的 、可以用readelf -S 命令确认。如果 有.debug_* 这些段名的话就是用 -g 编译的。

$ readelf -S a.out | grep '\.debug_'
  [29] .debug_aranges    PROGBITS        00000000 0cb8a6 002aa8 00      0   0  1
  [30] .debug_pubnames   PROGBITS        00000000 0ce34e 02d72c 00      0   0  1
  [31] .debug_info       PROGBITS        00000000 0fba7a 1670e5 00      0   0  1
  (略)
其他一些细微之处:

多线程程序也可以解析 (跟ltrace-0.5 不一样!)。
同样、fork的程序也可以解析。fork后的子进程也可以解析。
fork后exec执行也OK、exec执行的文件的符号会正确读取。
SIGSEGV/SIGILL/SIGBUS/SIGFPE等crash的程序也可以解析。crash的原因の命令の位置(EIP)也可以表示。
C++符号也可以demangle(可读化)后表示。
例子
这里举了五个用tracef解析程序的例子。
1. fork程序的解析
#include <stdio.h>
#include <sys/types.h>
#include <unistd.h>

int main() {
  if (fork() == 0) {
    printf("hello world\n");  
    return 1;
  }
  return 0;
}
用tracef解析以上程序的话会输出如下结果。fork后的进程自动开始解析。 --ff 选项可以把每个进程/线程的结果输出到Log文件。

$ ../src/tracef --synthetic -flATu ./fork
[pid 30133] 13:56:14.041015 +++ process 30133 attached (ppid 30132) +++
[pid 30133] 13:56:14.065944 === symbols loaded: './fork' ===
[pid 30133] 13:56:14.086854 ==> _start() at 0x080482e0
[pid 30133] 13:56:14.103301    ==> __libc_start_main@plt() at 0x080482a4
[pid 30133] 13:56:14.120804       ==> __libc_csu_init() at 0x08048410
[pid 30133] 13:56:14.142981          ==> _init() at 0x0804826c
[pid 30133] 13:56:14.167027             ==> call_gmon_start() at 0x08048304
[pid 30133] 13:56:14.198099             <== call_gmon_start() [eax = 0x0]
[pid 30133] 13:56:14.228514             ==> frame_dummy() at 0x08048390
[pid 30133] 13:56:14.256405             <== frame_dummy() [eax = 0x0]
[pid 30133] 13:56:14.287810             ==> __do_global_ctors_aux() at 0x08048480
[pid 30133] 13:56:14.316187             <== __do_global_ctors_aux() [eax = 0xffffffff]
[pid 30133] 13:56:14.346243          <== _init() [eax = 0xffffffff]
[pid 30133] 13:56:14.373957       <== __libc_csu_init() [eax = 0x80494e0]
[pid 30133] 13:56:14.400881       ==> main() at 0x080483b4 [/home/sato/tracef/sample/fork.c:6] 
[pid 30133] 13:56:14.424637          ==> fork@plt() at 0x080482c4
[pid 30134] 13:56:14.440226 +++ process 30134 attached (ppid 30133) +++
[pid 30134] 13:56:14.444849          <== fork@plt() [eax = 0x0]
[pid 30134] 13:56:14.455462          ==> puts@plt() at 0x080482b4
hello world
[pid 30134] 13:56:14.468085          <== puts@plt() [eax = 0xc]
[pid 30134] 13:56:14.495761       <== main() [eax = 0x1]
[pid 30134] 13:56:14.519362       ==> _fini() at 0x080484a8
[pid 30134] 13:56:14.539950          ==> __do_global_dtors_aux() at 0x08048330
[pid 30134] 13:56:14.566927          <== __do_global_dtors_aux() [eax = 0x0]
[pid 30134] 13:56:14.592884       <== _fini() [eax = 0x0]
[pid 30134] 13:56:14.616904 +++ process 30134 detached (ppid 30133) +++
[pid 30133] 13:56:14.625031          <== fork@plt() [eax = 0x75b6]
[pid 30133] 13:56:14.652768 --- SIGCHLD received (#17 Child exited) ---
[pid 30133] 13:56:14.660887       <== main() [eax = 0x0]
[pid 30133] 13:56:14.685255       ==> _fini() at 0x080484a8
[pid 30133] 13:56:14.701960          ==> __do_global_dtors_aux() at 0x08048330
[pid 30133] 13:56:14.726249          <== __do_global_dtors_aux() [eax = 0x0]
[pid 30133] 13:56:14.755024       <== _fini() [eax = 0x0]
[pid 30133] 13:56:14.781833 +++ process 30133 detached (ppid 30132) +++
2. exec程序的解析
写了下面一个程序它自身调用exec来做加法运算 (原来的代码是来自哪里的？...BinaryHacks? 记不起来了)。 $ ./exec 0 5 这样起动后，不可见的地方execve(2)反复执行来计算 5+4+3+2+1 、最后输出结果。argv[1]是累积的变量。

#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(int argc, char** argv) {
  if (argc != 3) return 0; /* usage: ./a.out 0 N */

  int accum = atoi(argv[1]);
  int n = atoi(argv[2]);

  if (n == 0) {
    printf("answer: %d\n", accum);
    return accum;
  }

  char p[32] = {0}, q[32] = {0};
  snprintf(p, 31, "%d", accum + n);
  snprintf(q, 31, "%d", n - 1);

  execlp("/proc/self/exe", "exe", p, q, NULL);
  return -1;
}
...程序内容暂且不提，如下所示，就算是有execve(2)的程序，处理内容也能被正确解析。

$ ../src/tracef --synthetic -flATuv ./exec 0 5   
[pid 30137] 13:59:40.506880 +++ process 30137 attached (ppid 30136) +++
[pid 30137] 13:59:40.516425 === symbols loaded: './exec' ===
[pid 30137] 13:59:40.523785 ==> _start() at 0x08048340
[pid 30137] 13:59:40.526590    ==> __libc_start_main@plt() at 0x080482e4
[pid 30137] 13:59:40.528794       ==> __libc_csu_init() at 0x08048550
[pid 30137] 13:59:40.533544          ==> _init() at 0x080482ac
[pid 30137] 13:59:40.536994             ==> call_gmon_start() at 0x08048364
[pid 30137] 13:59:40.540511             <== call_gmon_start() [eax = 0x0]
[pid 30137] 13:59:40.545888             ==> frame_dummy() at 0x080483f0
[pid 30137] 13:59:40.549673             <== frame_dummy() [eax = 0x0]
[pid 30137] 13:59:40.560435             ==> __do_global_ctors_aux() at 0x080485c0
[pid 30137] 13:59:40.585169             <== __do_global_ctors_aux() [eax = 0xffffffff]
[pid 30137] 13:59:40.618919          <== _init() [eax = 0xffffffff]
[pid 30137] 13:59:40.648524       <== __libc_csu_init() [eax = 0x8049638]
[pid 30137] 13:59:40.674937       ==> main(int argc <3>, POINTER argv <0xbff88a44>) at 0x08048414 [/home/sato/tracef/sample/exec.c:6] 
[pid 30137] 13:59:40.718245          ==> atoi@plt() at 0x08048314
[pid 30137] 13:59:40.744948          <== atoi@plt() [eax = 0x0]
[pid 30137] 13:59:40.770063          ==> atoi@plt() at 0x08048314
[pid 30137] 13:59:40.793048          <== atoi@plt() [eax = 0x5]
[pid 30137] 13:59:40.819975          ==> snprintf@plt() at 0x08048324
[pid 30137] 13:59:40.843943          <== snprintf@plt() [eax = 0x1]
[pid 30137] 13:59:40.873277          ==> snprintf@plt() at 0x08048324
[pid 30137] 13:59:40.898579          <== snprintf@plt() [eax = 0x1]
[pid 30137] 13:59:40.923149          ==> execlp@plt() at 0x080482f4
[pid 30137] 13:59:40.947893 === execve(2) called. reloading symbols... ===
[pid 30137] 13:59:40.962784 === symbols loaded: 'exe' ===
[pid 30137] 13:59:40.977023 ==> _start() at 0x08048340
...
[pid 30137] 13:59:42.522945          ==> execlp@plt() at 0x080482f4
[pid 30137] 13:59:42.546478 === execve(2) called. reloading symbols... ===
[pid 30137] 13:59:42.556684 === symbols loaded: 'exe' ===
[pid 30137] 13:59:42.568359 ==> _start() at 0x08048340
[pid 30137] 13:59:42.580952    ==> __libc_start_main@plt() at 0x080482e4
[pid 30137] 13:59:42.594888       ==> __libc_csu_init() at 0x08048550
[pid 30137] 13:59:42.607220          ==> _init() at 0x080482ac
[pid 30137] 13:59:42.616299             ==> call_gmon_start() at 0x08048364
[pid 30137] 13:59:42.625258             <== call_gmon_start() [eax = 0x0]
[pid 30137] 13:59:42.643251             ==> frame_dummy() at 0x080483f0
[pid 30137] 13:59:42.654002             <== frame_dummy() [eax = 0x0]
[pid 30137] 13:59:42.664041             ==> __do_global_ctors_aux() at 0x080485c0
[pid 30137] 13:59:42.673053             <== __do_global_ctors_aux() [eax = 0xffffffff]
[pid 30137] 13:59:42.688869          <== _init() [eax = 0xffffffff]
[pid 30137] 13:59:42.697093       <== __libc_csu_init() [eax = 0x8049638]
[pid 30137] 13:59:42.706019       ==> main(int argc <3>, POINTER argv <0xbf966364>) at 0x08048414 [/home/sato/tracef/sample/exec.c:6] 
[pid 30137] 13:59:42.719013          ==> atoi@plt() at 0x08048314
[pid 30137] 13:59:42.726840          <== atoi@plt() [eax = 0xf]
[pid 30137] 13:59:42.731108          ==> atoi@plt() at 0x08048314
[pid 30137] 13:59:42.735355          <== atoi@plt() [eax = 0x0]
[pid 30137] 13:59:42.738764          ==> printf@plt() at 0x08048304
answer: 15
[pid 30137] 13:59:42.745753          <== printf@plt() [eax = 0xb]
[pid 30137] 13:59:42.749255       <== main() [eax = 0xf]
[pid 30137] 13:59:42.752250       ==> _fini() at 0x080485e8
[pid 30137] 13:59:42.755001          ==> __do_global_dtors_aux() at 0x08048390
[pid 30137] 13:59:42.758079          <== __do_global_dtors_aux() [eax = 0x0]
[pid 30137] 13:59:42.767404       <== _fini() [eax = 0x0]
[pid 30137] 13:59:42.780894 +++ process 30137 detached (ppid 30136) +++
3. 递归
下面是末尾递归调用做加法运算的程序。根据是否有优化的不同，跟踪的结果输出也不同。除此之外，tar.gz里也包含执行相互递归 (mutual recursion) 的示例。

#include <stdio.h>

int sum(int n) {
  return n == 0 ? 0 : n + sum(n - 1);
}

int main() {
  int s = sum(10);
  printf("sum(10) = %d\n", s);
  return s;
}
首先是没有优化的执行文件的跟踪结果。递归被调用的过程如下所示，很容易理解。

$ ../src/tracef -lATv ./recursion
[pid 30102] +++ process 30102 attached (ppid 30101) +++
[pid 30102] === symbols loaded: './recursion' ===
[pid 30102] ==> _start() at 0x080482b0
[pid 30102]    ==> __libc_csu_init() at 0x08048410
[pid 30102]       ==> _init() at 0x08048250
[pid 30102]          ==> call_gmon_start() at 0x080482d4
[pid 30102]          <== call_gmon_start() [eax = 0x0]
[pid 30102]          ==> frame_dummy() at 0x08048360
[pid 30102]          <== frame_dummy() [eax = 0x0]
[pid 30102]          ==> __do_global_ctors_aux() at 0x08048480
[pid 30102]          <== __do_global_ctors_aux() [eax = 0xffffffff]
[pid 30102]       <== _init() [eax = 0xffffffff]
[pid 30102]    <== __libc_csu_init() [eax = 0x80494e4]
[pid 30102]    ==> main() at 0x080483b4 [/home/sato/tracef/sample/recursion.c:9] 
[pid 30102]       ==> sum(int n <10>) at 0x08048384 [/home/sato/tracef/sample/recursion.c:4] 
[pid 30102]          ==> sum(int n <9>) at 0x08048384 [/home/sato/tracef/sample/recursion.c:4] 
[pid 30102]             ==> sum(int n <8>) at 0x08048384 [/home/sato/tracef/sample/recursion.c:4] 
[pid 30102]                ==> sum(int n <7>) at 0x08048384 [/home/sato/tracef/sample/recursion.c:4] 
[pid 30102]                   ==> sum(int n <6>) at 0x08048384 [/home/sato/tracef/sample/recursion.c:4] 
[pid 30102]                      ==> sum(int n <5>) at 0x08048384 [/home/sato/tracef/sample/recursion.c:4] 
[pid 30102]                         ==> sum(int n <4>) at 0x08048384 [/home/sato/tracef/sample/recursion.c:4] 
[pid 30102]                            ==> sum(int n <3>) at 0x08048384 [/home/sato/tracef/sample/recursion.c:4] 
[pid 30102]                               ==> sum(int n <2>) at 0x08048384 [/home/sato/tracef/sample/recursion.c:4] 
[pid 30102]                                  ==> sum(int n <1>) at 0x08048384 [/home/sato/tracef/sample/recursion.c:4] 
[pid 30102]                                     ==> sum(int n <0>) at 0x08048384 [/home/sato/tracef/sample/recursion.c:4] 
[pid 30102]                                     <== sum() [eax = 0x0]
[pid 30102]                                  <== sum() [eax = 0x1]
[pid 30102]                               <== sum() [eax = 0x3]
[pid 30102]                            <== sum() [eax = 0x6]
[pid 30102]                         <== sum() [eax = 0xa]
[pid 30102]                      <== sum() [eax = 0xf]
[pid 30102]                   <== sum() [eax = 0x15]
[pid 30102]                <== sum() [eax = 0x1c]
[pid 30102]             <== sum() [eax = 0x24]
[pid 30102]          <== sum() [eax = 0x2d]
[pid 30102]       <== sum() [eax = 0x37]
[pid 30102]    <== main() [eax = 0x37]
[pid 30102]    ==> _fini() at 0x080484a8
[pid 30102]       ==> __do_global_dtors_aux() at 0x08048300
[pid 30102]       <== __do_global_dtors_aux() [eax = 0x0]
[pid 30102]    <== _fini() [eax = 0x0]
sum(10) = 55
[pid 30102] +++ process 30102 detached (ppid 30101) +++
下一次是优化后 (gcc -O1 -foptimize-sibling-calls) 的结果输出。函数调用 sum(10); 里被执行的加法运算已经看不到了。据此，跟踪优化后的二进制时需要注意一下。反之，也可以用tracef来调查优化是不是有效果。

$ ../src/tracef -lATv ./recursion_opt
[pid 30104] +++ process 30104 attached (ppid 30103) +++
[pid 30104] === symbols loaded: './recursion_opt' ===
[pid 30104] ==> _start() at 0x080482b0
[pid 30104]    ==> __libc_csu_init() at 0x080483f0
[pid 30104]       ==> _init() at 0x08048250
[pid 30104]          ==> call_gmon_start() at 0x080482d4
[pid 30104]          <== call_gmon_start() [eax = 0x0]
[pid 30104]          ==> frame_dummy() at 0x08048360
[pid 30104]          <== frame_dummy() [eax = 0x0]
[pid 30104]          ==> __do_global_ctors_aux() at 0x08048460
[pid 30104]          <== __do_global_ctors_aux() [eax = 0xffffffff]
[pid 30104]       <== _init() [eax = 0xffffffff]
[pid 30104]    <== __libc_csu_init() [eax = 0x80494c4]
[pid 30104]    ==> main() at 0x0804839c [/home/sato/tracef/sample/recursion.c:9] 
[pid 30104]       ==> sum(int n <10>) at 0x08048384 [/home/sato/tracef/sample/recursion.c:4] 
[pid 30104]       <== sum() [eax = 0x37]
[pid 30104]    <== main() [eax = 0x37]
[pid 30104]    ==> _fini() at 0x08048488
[pid 30104]       ==> __do_global_dtors_aux() at 0x08048300
[pid 30104]       <== __do_global_dtors_aux() [eax = 0x0]
[pid 30104]    <== _fini() [eax = 0x0]
sum(10) = 55
[pid 30104] +++ process 30104 detached (ppid 30103) +++
4. 多线程
使用pthread的程序也能被解析。

#include <stdio.h>
#include <pthread.h>

void* thread_entry(void* p) {
  printf("pthread_self()=%lu\n", pthread_self());
  return NULL;
}

int main() {
  pthread_t t;
  pthread_create(&t, NULL, thread_entry, 0);
  pthread_join(t, NULL);
  return 0;
}
tracef中的线程ID、和$ ps -L 输出的 "LWP" 数字或、生成线程的父线程的 /proc/pid/task/ 下面显示的数字一致。用pthread_self()获得的  unsigned long int值是不一样的值。容易混乱的点所以说明一下。

$ ../src/tracef  --synthetic -flT ./thread
[pid 30154] +++ process 30154 attached (ppid 30153) +++
[pid 30154] === symbols loaded: './thread' ===
[pid 30154] ==> _start() at 0x080483c0
[pid 30154]    ==> __libc_start_main@plt() at 0x08048380
[pid 30154]       ==> __libc_csu_init() at 0x08048520
[pid 30154]          ==> _init() at 0x08048338
[pid 30154]             ==> call_gmon_start() at 0x080483e4
[pid 30154]             <== call_gmon_start() [eax = 0x0]
[pid 30154]             ==> frame_dummy() at 0x08048470
[pid 30154]             <== frame_dummy() [eax = 0x0]
[pid 30154]             ==> __do_global_ctors_aux() at 0x08048590
[pid 30154]             <== __do_global_ctors_aux() [eax = 0xffffffff]
[pid 30154]          <== _init() [eax = 0xffffffff]
[pid 30154]       <== __libc_csu_init() [eax = 0x80495f8]
[pid 30154]       ==> main() at 0x080484b6 [/home/sato/tracef/sample/thread.c:11] 
[pid 30154]          ==> pthread_create@plt() at 0x080483a0
[pid 30155] +++ thread  30155 attached (ppid 30154) +++
[pid 30154]          <== pthread_create@plt() [eax = 0x0]
[pid 30154]          ==> pthread_join@plt() at 0x08048360
[pid 30155] ==> thread_entry() at 0x08048494 [/home/sato/tracef/sample/thread.c:5] 
[pid 30155]    ==> pthread_self@plt() at 0x080483b0
[pid 30155]    <== pthread_self@plt() [eax = 0xb7efcb90]
[pid 30155]    ==> printf@plt() at 0x08048390
pthread_self()=3085945744
[pid 30155]    <== printf@plt() [eax = 0x1a]
[pid 30155] <== thread_entry() [eax = 0x0]
[pid 30154]          <== pthread_join@plt() [eax = 0x0]
[pid 30154]       <== main() [eax = 0x0]
[pid 30154]       ==> _fini() at 0x080485b8
[pid 30154]          ==> __do_global_dtors_aux() at 0x08048410
[pid 30154]          <== __do_global_dtors_aux() [eax = 0x0]
[pid 30154]       <== _fini() [eax = 0x0]
[pid 30155] +++ thread  30155 detached (ppid 30154) +++
[pid 30154] +++ process 30154 detached (ppid 30153) +++
tracef: done
5. 大家喜闻乐见的抛出异常
异常抛用、栈打印的例子。

#include <stdio.h>

int c(int i) {
  if (i == 0) throw 0xff;
  return c(--i);
}

void b() { c(3); }

int a() {
  try { b(); } 
  catch(int& e) { return e; }
  return 0;
}

int main() {
  return a();
}
c()函数中参数 i = 0被调用的时刻抛出异常(__cxa_throw@plt)、到a()为止的栈信息被打印出来。a()返回值0xff也被明确地表示出来。C++的符号的demangle也可以执行。

$ ../src/tracef --synthetic -flT ./thread
[pid 30110] +++ process 30110 attached (ppid 30109) +++
[pid 30110] === symbols loaded: './throw' ===
[pid 30110] ==> _start() at 0x080484d0
[pid 30110]    ==> __libc_start_main@plt() at 0x08048458
[pid 30110]       ==> __libc_csu_init() at 0x08048680
[pid 30110]          ==> _init() at 0x08048420
[pid 30110]             ==> call_gmon_start() at 0x080484f4
[pid 30110]             <== call_gmon_start() [eax = 0x2e75d4]
[pid 30110]             ==> frame_dummy() at 0x08048580
[pid 30110]             <== frame_dummy() [eax = 0x0]
[pid 30110]             ==> __do_global_ctors_aux() at 0x080486f0
[pid 30110]             <== __do_global_ctors_aux() [eax = 0xffffffff]
[pid 30110]          <== _init() [eax = 0xffffffff]
[pid 30110]       <== __libc_csu_init() [eax = 0x804982c]
[pid 30110]       ==> main() at 0x0804864c [/home/sato/tracef/sample/throw.cpp:26] 
[pid 30110]          ==> a()() at 0x08048604 [/home/sato/tracef/sample/throw.cpp:16] 
[pid 30110]             ==> b()() at 0x080485f0 [/home/sato/tracef/sample/throw.cpp:11] 
[pid 30110]                ==> c(int)(int i <3>) at 0x080485a4 [/home/sato/tracef/sample/throw.cpp:5] 
[pid 30110]                   ==> c(int)(int i <2>) at 0x080485a4 [/home/sato/tracef/sample/throw.cpp:5] 
[pid 30110]                      ==> c(int)(int i <1>) at 0x080485a4 [/home/sato/tracef/sample/throw.cpp:5] 
[pid 30110]                         ==> c(int)(int i <0>) at 0x080485a4 [/home/sato/tracef/sample/throw.cpp:5] 
[pid 30110]                            ==> __cxa_allocate_exception@plt() at 0x08048468
[pid 30110]                            <== __cxa_allocate_exception@plt() [eax = 0x9e70058]
[pid 30110]                            ==> __cxa_throw@plt() at 0x08048478
[pid 30110]                            ==> __gxx_personality_v0@plt() at 0x080484a8
[pid 30110]                            <== __gxx_personality_v0@plt() [eax = 0x8]
...
[pid 30110]                            ==> __gxx_personality_v0@plt() at 0x080484a8
[pid 30110]                            <== __gxx_personality_v0@plt() [eax = 0x7]
[pid 30110]                            ==> __cxa_begin_catch@plt() at 0x08048498
[pid 30110]                            <== __cxa_begin_catch@plt() [eax = 0x9e70058]
[pid 30110]                            ==> __cxa_end_catch@plt() at 0x08048488
[pid 30110]                            <== __cxa_end_catch@plt() [eax = 0x0]
[pid 30110]          <== a()() [eax = 0xff]
[pid 30110]       <== main() [eax = 0xff]
[pid 30110]       ==> _fini() at 0x08048718
[pid 30110]          ==> __do_global_dtors_aux() at 0x08048520
[pid 30110]          <== __do_global_dtors_aux() [eax = 0x0]
[pid 30110]       <== _fini() [eax = 0x0]
[pid 30110] +++ process 30110 detached (ppid 30109) +++
6. main之前调用的函数
能观察到main函数被调用之后什么函数被调用。

#include <cstdio>
#include <cstddef>

// main之前被调用的三个函数

int foo() { return 1; }
int g = foo();

struct bar {
  bar() {}
  ~bar() throw() {}
} g2;

__attribute__((constructor))
void baz() {}

// main之前被初始化的变量value

template<const char* S, std::size_t L, std::size_t N = 0>
struct strSum_ {
  static const unsigned long value;
};

template<const char* S, std::size_t L, std::size_t N>
const unsigned long strSum_<S, L, N>::value = S[N] + strSum_<S, L, N + 1>::value; // XXX: runtime computation

template<const char* S, std::size_t L>
struct strSum_<S, L, L> {
  static const unsigned long value = 0;
};

// http://www.thescripts.com/forum/thread156880.html
template<typename T, std::size_t L> char (&lengthof_helper_(T(&)[L]))[L];
#define LENGTHOF(array) sizeof(lengthof_helper_(array))

extern const char s[] = "C++0x"; // external linkage 
int main() {
  return (int) strSum_<s, LENGTHOF(s) - 1>::value;
}
foo(), bar(), baz() 函数、 main()之前被调用、这些函数能够被跟踪出来。成员变量value的初始化也是在运行时发生（编译时不计算。这代码写的很烂大家可不要拷贝 ^^;）、这里根据函数的调用情况并不是都能被初始化，所以很遗憾这里不能跟踪。

$ ../src/tracef --plt -ClAT ./before_main2
[pid 17098] +++ process 17098 attached (ppid 17097) +++
[pid 17098] === symbols loaded: './before_main2' ===
[pid 17098] ==> _start() at 0x08048370
[pid 17098]    ==> __libc_start_main@plt() at 0x08048358
[pid 17098]       ==> __libc_csu_init() at 0x080485f0
[pid 17098]          ==> _init() at 0x08048310
[pid 17098]             ==> call_gmon_start() at 0x08048394
[pid 17098]             <== call_gmon_start() [eax = 0x2e75d4]
[pid 17098]             ==> frame_dummy() at 0x08048420
[pid 17098]             <== frame_dummy() [eax = 0x0]
[pid 17098]             ==> __do_global_ctors_aux() at 0x08048660
[pid 17098]                ==> global constructors keyed to _Z3foov() at 0x080485ac [/home/sato/tracef-trunk/sample/before_main2.cpp:44] 
[pid 17098]                   ==> __static_initialization_and_destruction_0(int, int)() at 0x08048482 [/home/sato/tracef-trunk/sample/before_main2.cpp:43] 
[pid 17098]                      ==> foo()() at 0x08048444 [/home/sato/tracef-trunk/sample/before_main2.cpp:6] 
[pid 17098]                      <== foo()() [eax = 0x1]
[pid 17098]                      ==> bar::bar()() at 0x080485c8 [/home/sato/tracef-trunk/sample/before_main2.cpp:10] 
[pid 17098]                      <== bar::bar()() [eax = 0x1]
[pid 17098]                      ==> __cxa_atexit@plt() at 0x08048338
[pid 17098]                      <== __cxa_atexit@plt() [eax = 0x0]
[pid 17098]                   <== __static_initialization_and_destruction_0(int, int)() [eax = 0x141]
[pid 17098]                   ==> baz()() at 0x0804844e [/home/sato/tracef-trunk/sample/before_main2.cpp:16] 
[pid 17098]                   <== baz()() [eax = 0x141]
[pid 17098]                <== global constructors keyed to _Z3foov() [eax = 0x141]
[pid 17098]             <== __do_global_ctors_aux() [eax = 0xffffffff]
[pid 17098]          <== _init() [eax = 0xffffffff]
[pid 17098]       <== __libc_csu_init() [eax = 0x80496bc]
[pid 17098]       ==> main() at 0x08048454 [/home/sato/tracef-trunk/sample/before_main2.cpp:41] 
[pid 17098]       <== main() [eax = 0x141]
[pid 17098]       ==> __tcf_0() at 0x0804846e [/home/sato/tracef-trunk/sample/before_main2.cpp:13] 
[pid 17098]          ==> bar::~bar()() at 0x080485ce [/home/sato/tracef-trunk/sample/before_main2.cpp:11] 
[pid 17098]          <== bar::~bar()() [eax = 0x804846e]
[pid 17098]       <== __tcf_0() [eax = 0x804846e]
[pid 17098]       ==> _fini() at 0x08048688
[pid 17098]          ==> __do_global_dtors_aux() at 0x080483c0
[pid 17098]          <== __do_global_dtors_aux() [eax = 0x0]
[pid 17098]       <== _fini() [eax = 0x0]
[pid 17098] +++ process 17098 detached (ppid 17097) +++
限制事项
交代目前知道的问题点:

使用表示行号的选项 -l 的话、消耗的内存比需要的多。处在调查中。
没有支持位置独立执行文件(PIE)。
是否是PIE、可用如下的方式使用readelf -h来dump ELF的头来确认。是ET_DYN的话PIE, ET_EXEC的话是普通的执行文件。file命令也可以区别，这里略过。
$ readelf -h pie_binary | grep Type:
  Type: DYN (Shared object file)
下面两个条件重复的情况下、throw的C++异常无法正常catch、进程SIGABRT退出(std::terminate)。如果要跟踪这样的throw&catch 的C++程序的话、请不要使用--plt 或 -T其中的一个选项。详细的内容请参照samples/throw3.cpp。
DSO里的C++异常抛出被执行、tracef跟踪的实行文件里该异常的catch被执行
追加--plt和-T两者到tracef来进行跟踪
用-Wl,-Bstatic和-Wl,-Bdynamic链接的执行文件、好像不能很好的处理。
命令行选项
推荐使用 tracef --plt -CflT 。

Usage:

  % tracef [option ...] command [arg ...]
  % tracef [option ...] -p pid

Options:

  -? [ --help ]               

     显示帮助

  -V [ --version ]            

     显示版本并退出

  -o [ --output ] arg  

     跟踪的结果不输出到stderr、而是输出到'arg'指定的文件里

  --ff

     记录到每个进程或线程单独的日志里。日志文件名是
     「用-o 指定的文件名」＋「进程/线程ID」。

  -f [ --trace-child ]

     用fork()或clone()生成的子进程、子线程也能跟踪。

  --synthetic
  --plt

     合成符号也成为跟踪的对象。使用该选项的话、库函数调用和系统调用
     （的一部分）也可以跟踪。例如、printf@plt() 和 signal@plt() 等等。

  -C [ --demangle ]

     将C++低级别的函数名变换成可读形式并表示。

  -t [ --time ]

     追加现在的时间到输出的各行。

  -u [ --microseconds ]  

     追加现在的时间（单位为微秒）到输出的各行。

  -A [ --arg ] 

     表示函数的参数名(EXPERIMENTAL)

  -v [ --arg-val ]

     表示函数的参数值(EXPERIMENTAL, 仅仅是x86)

  -T [ --call-tree ]  

     用树状图表示函数调用。解析对象是多进程/多进程的情况下、
     推荐同时使用-o 或 --ff 选项。

  --offset arg

     指定树形表示时函数调用的偏移量(空格数)。
     默认值是3。

  --no-pid 

     不表示进程ID。

  -i [ --no-eip ] 

     不表示函数的地址。

  -l [ --line-numbers ] 

     使用调试信息、表示函数被定义的文件和行号。

  -p [ --attach-process ] arg 

     attach 到进程ID为 'arg' 的进程。

  -X [ --exclude ] arg

     不跟踪 'arg'、无视。可以指定多个。指定arg为mangle后的符号。
实现机制
想写的时候写到日记里。下面是memo。

使用libbfd来读写解析対象的程序的symtab、可以获取所有函数的起始地址。顺带.plt上的合成(synthetic)符号也能获取。
通过libopcodes、从刚才得到的各个函数的起始地址、对二进制文件进行反汇编、通过查找ret命令来静态解析/特定函数的返回地址。虽然也有通过栈来动态实施的方法，这次采用的是静态的方法。一方面是想试试和ltrace不同的方法、另一方面多线程环境下动态设定的BP被别的线程访问的处理也比较麻烦。
查找ret/retq命令的时、_start() 函数不是用ret而是hlt、末尾递归函数中用-O2的话没有ret、throw退出的话没有ret、结构体值返回的函数 不是0xC3而是0xC2、__attribute((noreturn))的函数没有ret、调用这些函数的地方没有用call而是jmp、寻找的时候这些细微之处都要注意。
一个函数里存在多个ret。如使用switch时? v0.11开始查找所有的ret。如果没有好好查找的话，缩进会越来越深。
一个函数中、普通的leave-ret和、jmp到其他函数「起始位置」 (不是call) 可能共存。进入到这样的函数里jmp被执行的话、-T 的时候缩进比实际的深度加一。对策嘛.. 虽然能检测出「跳转到函数头部」的处理、还是不对应。这样的函数用-X选项来回避。或是用-O0来编译对应的对象这也没有问题。这样的代码用-O2编译的话好像也没问题@gcc-4.1.2/x86。详细情况参考 samples/hard_to_find_ret.c 和它的日志。
  switch(...) {
  case X:
     return hoge; // leave-ret になる
  default:
  }
  return fuga();  // 自分以外の関数を呼ぶ。jmp になりがち
} // 関数の終わり
检索.debug_info 、取得函数定义的文件名、行号的信息。检索同样的.debug_info、取得函数参数的信息。
fork解析対象后ptrace(TRACEME);然后setenv(LD_BIND_NOW=1);然后exec。如果不执行setenv的话、经由PLT的函数第一次调用时(_dl_runtime_resolve时)就会立即返回。但这样会延长子进程的启动时间可能不是很好的hack。
解析対象被加载到内存后、在函数的起始地址、RET命令地址设定断点。ptrace(POKE)仅仅是0xCC到这些地址上
解析対象执行0xCC时、信号会通知tracef。因此、tracef可表示适合的情息。在0xCC停止的解析対象重新开始的方法、和gdb一样(这里不加说明)。简言之返回EIP后返回原来的命令单步执行。详细的话参考此书「デバッガの理論と実装」。
PLT経由のジャンプのフックは、まずPLT先頭の jmp *0x80... のとこに0xCCを仕掛け、その次の命令 push 0x.. にも0xCCを仕掛けておく。jmp*を踏んで解析対象が止まったら、解析対象のスタック上のリターンアドレスをPEEKして、tracef側に記憶する。そのリターンアドレスをpush 0x..のアドレスにすりかえる(POKE)。push 0xにリターンしてくるとまた0xCCを踏んで止まるので、tracef側に記憶しておいた本物のリターンアドレスを、今度はEIPに書き込んでcontinue。以上。これはひどい。
この方法だと、DSO内で例外がthrowされて、それが解析対象のバイナリまで到達すると、abortする。ごめんなさいごめんなさいごめんなさい。
最低限の対策として、__cxa_throw@plt の先頭でブレークした時は、このリターンアドレスのtweakを行わないようにしておいた。これで、exe内でthrowして、exe内でcatchする場合は問題なくなった。dso内でthrowして、exe内でcatchするプログラムをtraceする場合は、--pltか-Tのどちらかをはずしてもらわないとダメだ...。やはりltrace風の(よくあるデバッガの)、スタック見てリターンアドレスに地雷置くやりかたのほうがよかったか。
GOT/PLT相关的代码是每个arch都有。x86_64和x86几乎一样、但是jmp* 是PC相对还是用#ifdef定义。
例外のthrowで表示(コールツリーのインデント具合)が乱れないよう、各プロセス/スレッドの関数呼び出し状況を、tracef側の std::stack<addr_t> にも記憶しておく。
命令行选项的解析、用boost::program_options。
谷歌了一下ptrace但没有什么文献、结果仅仅是依赖strace和ltrace的源代码。strace因为支持多OS变成魔窟、主要以ltrace为中心看代码。但是ltrace不支持线程不支持clone也不支持PTRACE_O_TRACECLONE。结果是尝试错误。然后、ltrace的 .dyn段的处理部分非常有意思，但是这次没有用到...
用strace去跟踪strace、用strace去跟踪ltrace许多地方都明白了。
プロセスのptraceを終了する(detach)と、そのプロセスが即死する現象に悩んだ。単に、プロセスの0xCCを元に戻さないままデタッチしていたからシグナルで死んでいるだけだった。ありがとうございました。forkすると0xCCなまま.textがコピーされることも忘れずに。当然ですが。
如果省略了传给ptrace(GETREGS)的结构体的初始化、那会造成valgrind出现大量的警告。之前虽然有注意可能是省略ptrace的参数导致的但还是花了一些时间。
ptrace(2)、以前strace也稍稍用过，但是尝试了许多方面后比想像的要有意思的多。早点玩玩就好了。
开发中感到乐趣无穷。

类似的工具
整理了能动态跟踪可执行文件自身的函数的工具的一览表。实现机制并不限定于的ptrace、有许多的方式。 基于ptrace的工具tracef、因为速度上比不上oprofile、可视化的功能上不敌callgrind、所以打算重点是放在实现一个和strace, ltrace 那样的容易使用的工具上。

CPU模拟器之类的工具
callgrind + kcachegrind
使用ptrace(2)的工具
ltrace的-x选项 (可以跟踪用-x指定的自身的函数。不能指定「全部」)
itrace (所有的处理用single step的工具)
xtrace (不能用make...。好像没有支持clone(2))
使用GCC -finstrument-functions 的工具
pvtrace
KLab的ftrace
etrace
glibc's xtrace
内核使用的工具
oprofile (虽然是个profiler但是call tree也能输出)
对源代码修改的工具
CTrace
traceString
没有调查
Frysk (由RedHat开发、用于Fedora标准版的工具)
fenris?
dude?
callgrind和oprofile确实很厉害。

TODO
PIE支持
配置文件 ~/.tracefrc
--exclude (-X) 选项的regex支持
--ff 時文件名的.pid前面、是否追加线程还是进程的用于识别的记号?
-l 时.debug_info重复读取
gettext支持
自己试着写一下使用ftrace代码的部分。然后、不用libdrawf 而是使用libdw (elfutils)。整型/指针类型以外的也支持。名字空间的支持、GNU C 的nested-function支持。
ptrace相关的部分、vfork支持、PTRACE_O_TRACESYSGOOD 支持、通过DR寄存器来确认singlestep是否成功 (x86)
ARM支持、x86_64支持、arch依赖部分的分离
arm Linux只能跑stub、tracef自身只能在PC上运行。
C++符号的demangler可能比较烂(使用libiberty。global constructors keyed to ... 的demangle等需要改善哪)
系统调用的表示? pretty print以外虽然比较简单。但pretty print比较困难、而且strace以上的东西搞不定还是不做了吧。
utrace支持(?)
x86_64上的-v支持、x86_64上用-m32编译的二进制文件用--plt跟踪发生错误的解决。
