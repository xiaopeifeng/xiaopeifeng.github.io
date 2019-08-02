---
layout: default
title: scatter-gather io
---

### 												scatter-gather io

##### what is scatter-gather io?

> *in computer science, vectored I/O also known as scatter/gather I/O, is a method of input and output by which a single procedure call sequentially reads data from multiple buffers and writes it to a single data stream, or read data from a data stream and write it to mutiple buffers, as defined in a vector of buffers. Scatter/gather refers to the process of gather data from, or scattering data into, the given set of buffers.*

​	the most common usage of this concept is the system call ***readv*** and ***writev***, if you are familar with network programming, you must have learned these two system call. 

##### what is the benefic of scatter-gather io?
scatter-gather io can make our code effiency and convience in some scenes expically while wring networking programming code. while optimizing our system program, two aspects will become the main roles after finishing framework, system processing data stream, algorithm optimizing work. one is try our best to decrease the system calls, and the other is avoid the memory copy. And scatter-gather io will help us on both side.

a simple example:

```c
struct header* hdr;
struct body*   body;
hdr = create_hear();
body = create_body();
```

the header and body are not contiguous blocks, but you want to send to other peer, what will you do next? now may be you have two chooses?

1. copying them into one block of memory using *memcpy*? and send it once
2. make two seperate system calls *write*

all these two methods can realize the purpose, but they aren't the best way. If you use write instead, code will be like

```c
struct iovec iov[3];
iov[0].iov_base = hdr;
iov[0].iov_len = sizeof(struct header);
iov[1].iov_base = body;
iov[1].iov_len = sizeof(struct body);
bytes = writev(fd, iov, sizeof(iov));
```

you found only one system call was used, and no *memcpy* was using, and also, these two buffers were written atomically like a single write operation.







