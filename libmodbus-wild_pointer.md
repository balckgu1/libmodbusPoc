A wild pointer causes program to crash



## libmodbus version

  v3.1.6

## OS and/or distribution

  Ubuntu22

## Environment

 ...

## Description

  After sending a specific message to the unit-test-server and running it for a while, ASAN detects a wild pointer.

## Actual behavior if applicable

  =================================================================
==8074==ERROR: AddressSanitizer: attempting free on address which was not malloc()-ed: 0x604000001234 in thread T0

## Expected behavior or suggestion

  everything is normal

## Steps to reproduce the behavior (commands or source code)

  Compile the program with ASAN

```
  ./autogen.sh
  ./configure --enable-static CC="clang -fsanitize=address  -O1 -g" CXX="clang -fsanitize=address  -O1 -g"
   make -j8
   clang -g -fsanitize=address -O1  unit-test-server.c -o unit-test-server -I ../src/ ../src/.libs/libmodbus.a
```
  run unit-test-server
 `./unit-test-server`

Send a message to unit-test-server using the python script in POC.zip
POC.zip:
[POC.zip](https://github.com/user-attachments/files/15981861/POC.zip)

`python3 socketv2.py -i 127.0.0.1 -p 1502 -f poc.txt`

Then, ctrl+c socketv2.py, you can see:

```
=================================================================
==8074==ERROR: AddressSanitizer: attempting free on address which was not malloc()-ed: 0x604000001234 in thread T0
    #0 0x725a12cb4537  (/lib/x86_64-linux-gnu/libasan.so.6+0xb4537)
    #1 0x6495a6f253e4  (/home/XXX/libmodbus/tests/unit-test-server+0x1c3e4)
    #2 0x6495a6f0f50b  (/home/XXX/libmodbus/tests/unit-test-server+0x650b)
    #3 0x725a12829d8f  (/lib/x86_64-linux-gnu/libc.so.6+0x29d8f)
    #4 0x725a12829e3f  (/lib/x86_64-linux-gnu/libc.so.6+0x29e3f)
    #5 0x6495a6f104e4  (/home/XXX/libmodbus/tests/unit-test-server+0x74e4)

Address 0x604000001234 is a wild pointer.
SUMMARY: AddressSanitizer: bad-free (/lib/x86_64-linux-gnu/libasan.so.6+0xb4537) 
==8074==ABORTING
Aborted (core dumped)
```

ASAN shows that the program attempted to call free() on an address that was not allocated using malloc(), which caused a crash. The specific message is given below:
Error type: attempted to call free() on an address not allocated using malloc().
Thread: The error occurred in the main thread (T0).
Address: 0x604000001234 is a "wild pointer", indicating that it is an invalid or uninitialized pointer.

By the way, other normal messages do not trigger this vulnerability, I guess this could be triggered due to the following reasons:
double free: the program called free() somewhere Twice dangling pointer: the pointer was not set to NULL after it was freed, causing it to be used again in subsequent code.
Uninitialized pointer: An uninitialized pointer was passed directly to free().
Memory out-of-bounds: Out-of-bounds accesses to arrays or buffers result in incorrect pointers being passed to free().