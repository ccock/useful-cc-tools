https://www.cnblogs.com/welhzh/p/4904874.html
https://wangfakang.github.io/c5/


c c++ 函数入口和出口的hook(gcc 编译选项)，然后打印出函数调用关系的方法
GCC Function instrumentation机制可以用来跟踪函数的调用关系，在gcc中对应的选项为“-finstrument-functions”。可查看gcc的man page来获取更详细信息。
编译时如果为gcc加上“-finstrument-functions”选项，那在每个函数的入口和出口处会各增加一个额外的hook函数的调用，增加的这两个函数分别为：

void __cyg_profile_func_enter (void *this_fn, void *call_site);
void __cyg_profile_func_exit  (void *this_fn, void *call_site);

其中第一个参数为当前函数的起始地址，第二个参数为返回地址，即caller函数中的地址。
这是什么意思呢？例如我们写了一个函数func_test()，定义如下：

static void func_test(v) {   /* your code... */}

那通过-finstrument-functions选项编译后，这个函数的定义就变成了：

static void func_test(v) {__cyg_profile_func_enter(this_fn, call_site);   /* your code... */  __cyg_profile_func_exit(this_fn, call_site);}

我们可以按照自己的需要去实现这两个hook函数，这样我们就可以利用this_fn和call_site这两个参数大做文章。
例如下面这段代码：
instrfunc.c:

#include <stdio.h>
#define DUMP(func, call) printf("%s: func = %p, called by = %p\n", __FUNCTION__, func, call)

void __attribute__((no_instrument_function)) __cyg_profile_func_enter(void *this_func, void *call_site) {DUMP(this_func, call_site);}
void __attribute__((no_instrument_function)) __cyg_profile_func_exit(void *this_func, void *call_site) {DUMP(this_func, call_site);}
int do_multi(int a, int b) {return a * b;}
int do_calc(int a, int b) {return do_multi(a, b);}

int main()
{
  int a = 4, b = 5;
  printf("result: %d\n", do_calc(a, b));  
  return 0;
}

这段代码中实现了两个hook函数，即打印出所在函数的函数地址以及返回地址。
编译代码：

[zhenfg@ubuntu]code:$ gcc -finstrument-functions instrfunc.c -o instrfunc
[zhenfg@ubuntu]code:$ ./instrfunc
__cyg_profile_func_enter: func = 0x8048521, called by = 0xb75554e3
__cyg_profile_func_enter: func = 0x80484d8, called by = 0x8048562
__cyg_profile_func_enter: func = 0x804849a, called by = 0x8048504
__cyg_profile_func_exit: func = 0x804849a, called by = 0x8048504
__cyg_profile_func_exit: func = 0x80484d8, called by = 0x8048562
result: 20
__cyg_profile_func_exit: func = 0x8048521, called by = 0xb75554e3

通过反汇编的代码（objdump -D instrfunc）可以看到，这些地址和函数的对应关系为：

__cyg_profile_func_enter: func = 0x8048521(main), called by = 0xb75554e3
__cyg_profile_func_enter: func = 0x80484d8(do_calc), called by = 0x8048562(main)
__cyg_profile_func_enter: func = 0x804849a(do_multi), called by = 0x8048504(do_calc)
__cyg_profile_func_exit: func = 0x804849a(do_multi), called by = 0x8048504(do_calc)
__cyg_profile_func_exit: func = 0x80484d8(do_calc), called by = 0x8048562(main)
result: 20
__cyg_profile_func_exit: func = 0x8048521(main), called by = 0xb75554e3

实际上这就给出了函数的调用关系。

如果不想跟踪某个函数，可以给该函数指定“no_instrument_function”属性。需要注意的是，__cyg_profile_func_enter()和__cyg_profile_func_exit()这两个hook函数是一定要加上“no_instrument_function”属性的，不然，自己跟踪自己就会无限循环导致程序崩溃，当然，也不能在这两个hook函数中调用其他需要被跟踪的函数。

得到一系列的地址看起来不太直观，我们更希望看到函数名，幸运的是，addr2line工具为我们提供了这种可能。我们先看一下addr2line的使用方法：

[zhenfg@ubuntu]code:$ addr2line --helpUsage:
addr2line [option(s)] [addr(s)] Convert addresses into line number/file name pairs. If no addresses are specified on the command line, they will be read from stdin The options are:  
@<file>                Read options from <file>  
-a --addresses         Show addresses  
-b --target=<bfdname>  Set the binary file format  
-e --exe=<executable>  Set the input file name (default is a.out)  
-i --inlines           Unwind inlined functions  
-j --section=<name>    Read section-relative offsets instead of addresses  
-p --pretty-print      Make the output easier to read for humans  
-s --basenames         Strip directory names  
-f --functions         Show function names  
-C --demangle[=style]  Demangle function names  
-h --help              Display this information  
-v --version           Display the program's version

首先要注意，使用addr2line工具时，需要用gcc的“-g”选项编译程序增加调试信息。
同样是上面的程序，我们加上-g选项再编译一次：

[zhenfg@ubuntu]code:$ gcc -g -finstrument-functions instrfunc.c -o instrfunc
[zhenfg@ubuntu]code:$ ./instrfunc
__cyg_profile_func_enter: func = 0x8048521, called by = 0xb757d4e3
__cyg_profile_func_enter: func = 0x80484d8, called by = 0x8048562
__cyg_profile_func_enter: func = 0x804849a, called by = 0x8048504
__cyg_profile_func_exit: func = 0x804849a, called by = 0x8048504
__cyg_profile_func_exit: func = 0x80484d8, called by = 0x8048562
result: 20
__cyg_profile_func_exit: func = 0x8048521, called by = 0xb757d4e3

使用addr2line尝试查找0x8048504地址所在的函数：

[zhenfg@ubuntu]code:$ addr2line -e instrfunc -a 0x8048504 -fp -s
0x08048504: do_calc at instrfunc.c:25

这样一来，就可以通过gcc的“-finstrument-functions”选项结合addr2line工具，方便的对一个程序中的函数进行跟踪。并且既然我们可以自己实现hook函数，那不仅仅可以用来跟踪函数调用关系，你可以在hook函数中添加自己想做的事情，例如添加一些统计信息。
另外，我们知道__builtin_return_address(level)宏可以获得不同层级的函数返回地址，但是在某些体系架构（如mips）中，__builtin_return_address(level)只能获得当前函数的直接调用者的地址，即level只能是0，那这时，就可使用上述方法来跟踪函数调用关系（mips中竟然能用，确实有些小吃惊）。对于上面的例子，如果需要打印调用关系，只需要将对应的行替换成：

#define DUMP(func, call) printf("%p\n", __builtin_return_address(0))

或

#define DUMP(func, call) printf("%p\n", call_site)                     // 建议用这个，这个可以看到调用者实施调用的具体行号

接下来可以看一下gcc是如何将hook函数嵌入各个函数中的，以反汇编代码中的do_multi()函数为例（这是mips的汇编代码），在mips中，ra寄存器用来存储返回地址，a0-a3用来做函数参数。
004006c8 <do_multi>:  
4006c8:   27bdffd8    addiu   sp,sp,-40  
4006cc:  afbf0024    sw  ra,36(sp)   ;;存储ra寄存器（返回地址）的值  
4006d0: afbe0020    sw  s8,32(sp)  
4006d4:  afb1001c    sw  s1,28(sp)  
4006d8:  afb00018    sw  s0,24(sp)  
4006dc:  03a0f021    move    s8,sp  
4006e0:  03e08021    move    s0,ra   ;;s0 = ra  
4006e4:  afc40028    sw  a0,40(s8)  
4006e8:  afc5002c    sw  a1,44(s8)  
4006ec:  02001021    move    v0,s0   ;;v0 = s0  
4006f0:  3c030040    lui v1,0x40  
4006f4:    246406c8    addiu   a0,v1,1736  ;;将本函数的地址赋值给a0寄存器  
4006f8: 00402821    move    a1,v0       ;;将返回地址ra的值赋值给a1寄存器  
4006fc:   0c100188    jal 400620 <__cyg_profile_func_enter> ;;调用hook函数  
400700:   00000000    nop  
400704:    8fc30028    lw  v1,40(s8)  
400708:  8fc2002c    lw  v0,44(s8)  
40070c:  00000000    nop  
400710:    00620018    mult    v1,v0  
400714:  00008812    mflo    s1  
400718: 02001021    move    v0,s0  
40071c:  3c030040    lui v1,0x40  
400720:    246406c8    addiu   a0,v1,1736  ;;将本函数的地址赋值给a0寄存器  
400724: 00402821    move    a1,v0       ;;将返回地址ra的值赋值给a1寄存器  
400728:   0c10019d    jal 400674 <__cyg_profile_func_exit> ;;调用hook函数  
40072c:    00000000    nop  
400730:    02201021    move    v0,s1  
400734:  03c0e821    move    sp,s8  
400738:  8fbf0024    lw  ra,36(sp)   ;;恢复ra寄存器（返回地址）的值  
40073c: 8fbe0020    lw  s8,32(sp)  
400740:  8fb1001c    lw  s1,28(sp)  
400744:  8fb00018    lw  s0,24(sp)  
400748:  27bd0028    addiu   sp,sp,40  
40074c:   03e00008    jr  ra  
400750: 00000000    nop

上述反汇编的代码中，使用“-finstrument-functions”选项编译程序所增加的指令都已注释出来，实现没什么复杂的，在函数中获得自己的地址和上一级caller的地址并不是什么难事，然后将这两个地址传给__cyg_profile_func_enter和__cyg_profile_func_exit就好了。

 

 

还有一种更好的方法是直接打印出函数的名字(预编译时加上 -D_GNU_SOURCE， 链接时加上 -ldl ，android ndk-build 似乎不用加入-D_GNU_SOURCE，要不要加入-D_GNU_SOURCE自己可以去看dlfcn.h的源码) (如果dli_sname为NULL，加上参数 -Wl,--export-dynamic 试试)：

instrfunc.c
复制代码
#include <stdio.h>
#include <stdlib.h>
#include <dlfcn.h>

// #define DUMP(func, call) printf("%s: func = %p, called by = %p\n", __FUNCTION__, func, call)
#define DUMP(func, call) printf("%p\n", call_site)

static int call_level = 0;
static void *last_fn = NULL;

void __attribute__((no_instrument_function)) __cyg_profile_func_enter(void *this_func, void *call_site)
{
    // DUMP(this_func, call_site);
    int i;
    Dl_info di;

    if (last_fn != this_func) ++call_level;
    for (i = 0; i < call_level-1; i++) printf("   ");
    if (dladdr(this_func, &di)) {
        printf("%s\t\t(%s)\n", di.dli_sname ? di.dli_sname : "<unknown>", di.dli_fname);
    }
    last_fn = this_func;
}

void __attribute__((no_instrument_function)) __cyg_profile_func_exit(void *this_func, void *call_site)
{
    // DUMP(this_func, call_site);

    --call_level;
#if 0
    Dl_info di;
    if (dladdr(this_func, &di)) {
        printf("%s (%s)\n", di.dli_sname ? di.dli_sname : "<unknown>", di.dli_fname);
    }
#endif
}

int do_multi(int a, int b) {return a * b;}
int do_add(int a, int b) {return a+b;}
int do_calc(int a, int b) {do_multi(a, b);}

int main()
{
  int a = 4, b = 5;
  do_calc(a, b);
  do_add(a, b);
  return 0;
}
复制代码
编译方式(如果应用里包含线程，可以打印出tid，这样就能把不同线程里的调用关系筛选出来)：

gcc -g -D_GNU_SOURCE -finstrument-functions instrfunc.c -o instrfunc -ldl -Wl,--export-dynamic
输出：

main        (./instrfunc)
   do_calc        (./instrfunc)
      do_multi        (./instrfunc)
   do_add        (./instrfunc)
 

应用里面带thread的打印代码：

复制代码
#include <stdio.h>
#include <stdlib.h>
#include <dlfcn.h>
#include <pthread.h>

// #define DUMP(func, call) printf("%s: func = %p, called by = %p\n", __FUNCTION__, func, call)
#define DUMP(func, call) printf("%p\n", call_site)

static int call_level = 0;
static void *last_fn = NULL;

void __attribute__((no_instrument_function)) __cyg_profile_func_enter(void *this_func, void *call_site)
{
    // DUMP(this_func, call_site);
    int i;
    Dl_info di;

    for (i = 0; i < call_level; i++) printf("   ");
    if (last_fn != this_func) ++call_level;
    if (dladdr(this_func, &di)) {
        printf("%s\t\t(%s)\ttid:%lu -->\n", di.dli_sname ? di.dli_sname : "<unknown>", di.dli_fname, pthread_self());
        // printf("%s\t\t(%s)\ttid:%lx %lu\n", di.dli_sname ? di.dli_sname : "<unknown>", di.dli_fname, pthread_self(), pthread_self());
        // printf("%s\t\t(%s)\ttid:%x\n", di.dli_sname ? di.dli_sname : "<unknown>", di.dli_fname, (int) (0xFFFFFF & pthread_self()));
    }
    last_fn = this_func;
}

void __attribute__((no_instrument_function)) __cyg_profile_func_exit(void *this_func, void *call_site)
{
    // DUMP(this_func, call_site);

    --call_level;
#if 0
    Dl_info di;
    int i;
    for (i = 0; i < call_level; i++) printf("   ");
    if (dladdr(this_func, &di)) {
        printf("%s\t\t(%s)\ttid:%lu <--\n", di.dli_sname ? di.dli_sname : "<unknown>", di.dli_fname, pthread_self());
    }
#endif
}

int do_multi(int a, int b) {return a * b;}
int do_add(int a, int b) {return a+b;}
int do_calc(int a, int b) {do_multi(a, b);}

void func_c(void) {}
void func_b(void) {func_c();}
void func_a(void) {func_b(); func_c();}
void *thread_func_a(void *para) {func_a();}
void *thread_func_b(void *para) {func_a();}

int main()
{
  int a = 4, b = 5;
  do_calc(a, b);
  pthread_t pta, ptb;
  pthread_create(&pta, NULL, thread_func_a, (void *) NULL);
  do_add(a, b);
  pthread_create(&ptb, NULL, thread_func_b, (void *) NULL);
  pthread_join(pta, NULL);
  pthread_join(ptb, NULL);
  return 0;
}
复制代码
编译方式：

gcc -g -D_GNU_SOURCE -finstrument-functions instrfunc.c -o instrfunc -ldl -lpthread -Wl,--export-dynamic
运行结果：

复制代码
main        (./instrfunc)    tid:140079528912704 -->
   do_calc        (./instrfunc)    tid:140079528912704 -->
      do_multi        (./instrfunc)    tid:140079528912704 -->
   do_add        (./instrfunc)    tid:140079528912704 -->
   thread_func_a        (./instrfunc)    tid:140079518557952 -->
      func_a        (./instrfunc)    tid:140079518557952 -->
         func_b        (./instrfunc)    tid:140079518557952 -->
            func_c        (./instrfunc)    tid:140079518557952 -->
         func_c        (./instrfunc)    tid:140079518557952 -->
thread_func_b        (./instrfunc)    tid:140079510165248 -->
   func_a        (./instrfunc)    tid:140079510165248 -->
      func_b        (./instrfunc)    tid:140079510165248 -->
         func_c        (./instrfunc)    tid:140079510165248 -->
      func_c        (./instrfunc)    tid:140079510165248 -->
复制代码
如果要在android ndk下将以上两个hook函数编译成库(最好编译成静态库)，需要在Application.mk里包含：

APP_OPTIM := debug
APP_ABI   := armeabi-v7a
APP_CFLAG := -g -ggdb -O0
APP_PLATFORM := android-8
 

 

 

感谢：

https://groups.google.com/forum/#!topic/gnu.gcc.help/a-hvguqe10I

 

添加include：

+        #include <dlfcn.h>

在__cyg_profile_func_ente（）里添加：

+         Dl_info di;

+        fprintf(fp,  "entering %p", (int *)this_fn);
+        if (dladdr(this_fn, &di)) {
+          fprintf(fp,  " %s (%s)", di.dli_sname ? di.dli_sname : "<unknown>", di.dli_fname);
+        }
+        fputs("\n", fp);

 

 

No: the addresses that you print depend on where the in memory the
library is loaded, and that address can (and does on newer Linux
distributions) change from one run to the next.

Since nm can't possibly know where the library will be loaded,
it follows that nm can't possibly print the same address.

But when your __cyg_profile_func_enter is executing, it *can*
ask dynamic loader for the base address, and print the address that
will match output from nm.

Even better, it can simply ask dynamic loader for the name, and
print *that*, so you wouldn't have to "match" by hand at all.

Here is the code that does that. I only did the 'entering' part:

$ diff -u tracer.c.orig tracer.c
--- tracer.c.orig       2007-12-04 20:09:08.969400312 -0800
+++ tracer.c    2007-12-04 20:06:33.036105776 -0800
@@ -7,6 +7,7 @@


         #include <sys/types.h>
         #include <sys/stat.h>
         #include <unistd.h>
+        #include <dlfcn.h>


 
         void __cyg_profile_func_enter(void *this_fn, void *call_site) __attribute__((no_instrument_function));
         void __cyg_profile_func_exit(void *this_fn, void *call_site) __attribute__((no_instrument_function));
@@ -18,12 +19,17 @@


 void * last_fn;
 void __cyg_profile_func_enter(void *this_fn, void *call_site)
 {
+        Dl_info di;


         if (fp == NULL) fp = fopen( "trace.txt", "w" );
         if (fp == NULL) exit(-1);
 
         if ( this_fn!=last_fn) ++call_level;
         for (int i=0;i<=call_level;i++) fprintf(fp,"\t");
-        fprintf(fp,  "entering %p\n", (int *)this_fn);
+        fprintf(fp,  "entering %p", (int *)this_fn);
+        if (dladdr(this_fn, &di)) {
+          fprintf(fp,  " %s (%s)", di.dli_sname ? di.dli_sname : "<unknown>", di.dli_fname);
+        }
+        fputs("\n", fp);
         (void)call_site;
         last_fn = this_fn;


You'll need to link main with -ldl.
And here is a snipet of resulting output:

                entering 0x80488d0 __gxx_personality_v0 (./main)
                        entering 0xd4fade _ZN4test9function3Ec (./libtest.so)
                                entering 0xd4fa5a _ZN4test9function2Ei (./libtest.so)
                                        entering 0xd4f9e4 _ZN4test9function1El (./libtest.so)
                                exiting 0xd4f9e4

If you want demangled names, there is a function for that as well
(cplus_demangle() in libiberty).

 

 

下面是别人打印到文件且包含调用层级的方法（自己按照上面的打印函数名称的添加下就行）：

https://groups.google.com/forum/#!topic/gnu.gcc.help/a-hvguqe10I

I have the following c++ code in 4 files: test.h, test.cpp, main.cpp,
tracer.c

<test.h>
class test {
    int function1 (long l);
    int function2 (int i);
  public:
    int function3 (char c);

};

<test.cpp>
#include <iostream>
#include "test.h"
using namespace std;

int test::function1 (long l) {
cout << "in function 1" << endl;
  return l;
}

int test::function2 (int i) {
cout << "in function 2" << endl;
  return test::function1 (i) + 1;
}

int test::function3 (char c) {
cout << "in function 3" << endl;
  return test::function2 (c) + 1;

}

<main.cpp>
#include <iostream>
#include "test.h"
using namespace std;

int main () {
  class test *ptest;
  ptest = new test();
  return ptest->function3 (1);

}

<tracer.c>
#ifdef __cplusplus
extern "C"
{

        #include <stdio.h>
        #include <stdlib.h>
        #include <sys/types.h>
        #include <sys/stat.h>
        #include <unistd.h>

        void __cyg_profile_func_enter(void *this_fn, void *call_site)
__attribute__((no_instrument_function));
        void __cyg_profile_func_exit(void *this_fn, void *call_site)
__attribute__((no_instrument_function));

}

#endif
static FILE *fp;
int call_level=0;
void * last_fn;
void __cyg_profile_func_enter(void *this_fn, void *call_site)
{
        if (fp == NULL) fp = fopen( "trace.txt", "w" );
        if (fp == NULL) exit(-1);

        if ( this_fn!=last_fn) ++call_level;
        for (int i=0;i<=call_level;i++) fprintf(fp,"\t");
        fprintf(fp,  "entering %p\n", (int *)this_fn);
        (void)call_site;
        last_fn = this_fn;

}

void __cyg_profile_func_exit(void *this_fn, void *call_site)
{
        --call_level;
        for (int i=0;i<=call_level;i++) fprintf(fp,"\t");
        fprintf(fp, "exiting %p\n", (int *)this_fn);
        (void)call_site;

}

Next I build using the following commands:

g++ -fPIC -g -finstrument-functions -c test.cpp tracer.c
g++ -shared -Wall,soname,libtest.so.0 -o libtest.so.0.0 libtest.o
tracer.o
ln -sf libtest.so.0.0 libtest.so
setenv LD_LIBRARY_PATH ${LD_LIBRARY_PATH}:.
g++ -g -finstrument-functions -o main -ltest -L./ main.cpp

when I run ./main I get the following output from the
__cyg_profile_func_enter and __cyg_profile_func_exit funtions
