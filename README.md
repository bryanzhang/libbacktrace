backtrace是个常见的库，可以输出堆栈详细信息，包括源文件名以及行号，但原来做法直接使用malloc/free不可重入。
小改动让libbacktrace可重入，使用newlib的_malloc_r和_free_r方法，从而确保可在信号处理时调用。

另外暴露backtrace_unwind(struct _Unwind_Context*, struct backtrace_data*)，从而允许在打印出指定时刻、指定线程的堆栈，从而方便多线程的堆栈输出。

原仓库：https://github.com/ianlancetaylor/libbacktrace.git

# 安装:
```
  sudo apt install libnewlib-dev && ./configure && make && sudo make install # ubuntu 22.04
```

# 测试使用
test.cc:
```
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <signal.h>
#include <backtrace.h>

/*

  A libbacktrace example program.

  libbacktrace is found at https://github.com/ianlancetaylor/libbacktrace

  Compile this file with: gcc -g -o test test.c -lbacktrace

*/

static void error_callback (void *data, const char *message, int error_number) {
  if (error_number == -1) {
    fprintf(stderr, "If you want backtraces, you have to compile with -g\n\n");
    exit(1);
  } else {
    fprintf(stderr, "Backtrace error %d: %s\n", error_number, message);
  };
};

static int full_callback (void *data, uintptr_t pc, const char *pathname, int line_number, const char *function) {
  static int unknown_count = 0;
  if (pathname != NULL || function != NULL || line_number != 0) {
    if (unknown_count) {
      fprintf(stderr, "    ...\n");
      unknown_count = 0;
    };
    const char *filename = rindex(pathname, '/');
    if (filename) filename++; else filename = pathname;
    fprintf(stderr, "  %s:%d -- %s\n", filename, line_number, function);
  } else {
    unknown_count++;
  };
  return 0;
};

struct backtrace_state *state;

void sigsegv_handler (int number) {
  fprintf(stderr, "\n*** Segmentation Fault ***\n\n");
  backtrace_full(state, 0, full_callback, error_callback, NULL);
  printf("\n");
  //backtrace_print(state, 0, stderr);
  exit(1);
};

void function_two () {
  *((void **) 0) = 0; // program crashes here
};

void function_one () {
  function_two();
};

int main () {
  signal(SIGSEGV, sigsegv_handler);
  state = backtrace_create_state(NULL, 0, error_callback, NULL);
  function_one();
  return 0;
};
```
编译：
```
g++ test.cc -g -lbacktrace -o a.out
```
运行：
```
./a.out
```
输出：
```
*** Segmentation Fault ***

  st.cc:46 -- _Z15sigsegv_handleri
  sigaction.c:0 -- (null)
  st.cc:53 -- _Z12function_twov
  st.cc:57 -- _Z12function_onev
  st.cc:63 -- main
  libc-start.c:308 -- __libc_start_main
```
# libbacktrace
A C library that may be linked into a C/C++ program to produce symbolic backtraces

Initially written by Ian Lance Taylor <iant@golang.org>.

This is version 1.0.
It is likely that this will always be version 1.0.

The libbacktrace library may be linked into a program or library and
used to produce symbolic backtraces.
Sample uses would be to print a detailed backtrace when an error
occurs or to gather detailed profiling information.
In general the functions provided by this library are async-signal-safe,
meaning that they may be safely called from a signal handler.

The libbacktrace library is provided under a BSD license.
See the source files for the exact license text.

The public functions are declared and documented in the header file
backtrace.h, which should be #include'd by a user of the library.

Building libbacktrace will generate a file backtrace-supported.h,
which a user of the library may use to determine whether backtraces
will work.
See the source file backtrace-supported.h.in for the macros that it
defines.

As of October 2020, libbacktrace supports ELF, PE/COFF, Mach-O, and
XCOFF executables with DWARF debugging information.
In other words, it supports GNU/Linux, *BSD, macOS, Windows, and AIX.
The library is written to make it straightforward to add support for
other object file and debugging formats.

The library relies on the C++ unwind API defined at
https://itanium-cxx-abi.github.io/cxx-abi/abi-eh.html
This API is provided by GCC and clang.
