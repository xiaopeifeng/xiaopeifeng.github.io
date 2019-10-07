---
layout: default
title: why libcurl coredump under no networking environment?
---

### why libcurl coredump under no networking environment?

libcurl is often used by c programmer as a http client library, and I found a client program coredump happend under no network condition. the program is depending on libcurl, and I try to  use gdb to watch stack frame info after  coredump happened, but i reap nothing. After a longtime watching and analyse the  running log, I found some unimaginable fact, that thread id was changed. this program is consist of multiple thread, and the worker thread do some http communacation with server side under the help of libcurl. the worker id was changged every time coredump happened, and this is the only clue I found. Stack overflow? I try to increase the thread stack size, but it does not help. No coredump file, No clue, I have no such experience, so i can only get help from Google.

After search with some keywords with Google, I get some detail info about libcurl. libcurl has three type DNS resolve way, and my program is running in embedding device, and the version of libcurl was a little old, so used the default way, Block DNS resolving. and if the Block time was elapsed, the Block status will be interrupted by  ALARM signal.

i went on readding this part of code in libcurl source code, it use sigsetjmp/siglongjmp and signal ALARM to implement the Block DNS Resolving. But under multiple threads environment, the signal handler function is most likely called in main thread, so if siglongjmp was called in main thread, and sigsetjmp was called in worker thread, what will happen?

the saved stack info by sigsetjmp will be executed by main thread, and this explained the phenomenon of "thread id was changed". After I recompile libcurl with thread-pool DNS Resolving, coredump problem was solved.

I also write a code snipet to explain the main idea of libcurl blocking-dns-resolving.

```c
#include <setjmp.h>
#include <stdio.h>
#include <stdlib.h>
#include <signal.h>

#include <sys/types.h>
#include <sys/syscall.h>
#include <pthread.h>
#include <unistd.h>


#define gettid syscall(SYS_gettid)

sigjmp_buf jmp_ctx;
static int number = 10;

void
alarm_func(int n) {
  printf("%ld enter alarm_func\n", gettid);
  number++;
  siglongjmp(jmp_ctx, 0);
}

void
longtimetask(int time_used) {
  printf("%ld long time task running\n", gettid);
  sleep(time_used);
  alarm(0);
}

void*
worker(void* args) {
  printf("enter worker thread: %ld\n", gettid);
  signal(SIGALRM, alarm_func);
  alarm(5);

  int res = sigsetjmp(jmp_ctx, 0);
  printf("tid: %ld sigsetjmp return value: %d\n", gettid, res);
  printf("tid: %ld number: %d\n", gettid, number);
  if (res) {
    goto fail;
  }

  longtimetask(atoi((char*)args));
  goto ok;

fail:
  printf("tid: %ld worker fail\n", gettid);
  return NULL;

ok:
  printf("tid: %ld worker ok\n", gettid);
  return;
}

void
main_jobs() {
  long count = 30;
  while (count--) {
    printf("tid: %ld, count %ld\n", gettid, count);
    sleep(1);
  }
}

int main(int argc, char **argv) {
  if (argc != 2) {
    printf("usage: %s time\n", argv[0]);
    exit(0);
  }

  printf("start worker thread!\n");

  pthread_t th;
  pthread_create(&th, NULL, worker, argv[1]);

  main_jobs();

  pthread_join(th, NULL);

  printf("main thread exit!\n");
}
```







