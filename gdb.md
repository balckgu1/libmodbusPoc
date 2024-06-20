## libmodbus version

libmodbus v3.1.6

## OS and/or distribution

Ubuntu 18

## Description

In modbus.c, the value of the mb_mapping->tab_registers pointer may have been accidentally modified after it was allocated by malloc() so that it no longer points to the address where it was first allocated, resulting in a double-free error when executing the free(mb_mapping->tab_registers) statement in modbus_mapping_free(), which can cause the server to terminate unexpectedly. registers) statement in modbus_mapping_free() generates a double-free error, which can cause the server to terminate unexpectedly. ASAN summarizes this as a buffer overflow vulnerability.

## Actual behavior if applicable

double free or corruption (out)

## Vulnerability Analysis in GDB

### 1. First, breakpoints at line 196 of unit-test-server.c and line 1867 of modbus.c.

~~~
break unit-test-server.c:196
break modbus.c:1867
~~~

### 2. Then, run the program using the POC

~~~
(gdb) run < ./poc
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
[New Thread 0x7ffff7bff640 (LWP 3956)]
[New Thread 0x7ffff73fe640 (LWP 3957)]
[Thread 0x7ffff7bff640 (LWP 3956) exited]
The client connection from 0.0.0.0 is accepted
Waiting for an indication...
<00><00><00><00><00><0D><FF><03><01><62><00><01>
[00][00][00][00][00][05][FF][03][02][00][00]
Waiting for an indication...
<00><00><00><00><00><0D><FF><17><01><62><00><10><01><5D><00><01><02><AA><BD>
[00][00][00][00][00][23][FF][17][20][00][00][00][00][00][00][00][00][00][00][00][00][00][00][00][00][00][00][00][00][00][00][00][00][00][00][00][00][00][00][00][00]
Waiting for an indication...

ERROR Connection reset by peer: read

<00><01><02>Quit the loop: Connection reset by peer

Thread 1 "unit-test-serve" hit Breakpoint 1, main (argc=<optimized out>, argv=<optimized out>) at unit-test-server.c:196
196	    modbus_mapping_free(mb_mapping);
~~~

### 3. Execute the code in a single step and check the value of the pointer at the second breakpoint.

~~~
(gdb) c
Continuing.

Thread 1 "unit-test-serve" hit Breakpoint 3, modbus_mapping_free (mb_mapping=mb_mapping@entry=0x555555578430) at modbus.c:1867
1867	    free(mb_mapping->tab_input_registers);
(gdb) info args 
mb_mapping = 0x555555578430
(gdb) info mb_mapping
Undefined info command: "mb_mapping".  Try "help info".
(gdb) p mb
No symbol "mb" in current context.
(gdb) p mb_mapping 
$1 = (modbus_mapping_t *) 0x555555578430
(gdb) p *mb_mapping 
$2 = {nb_bits = 37, start_bits = 304, nb_input_bits = 22, 
  start_input_bits = 452, nb_input_registers = 1, start_input_registers = 264, 
  nb_registers = 32, start_registers = 352, tab_bits = 0x555555578480 "", 
  tab_input_bits = 0x5555555784b0 "", tab_input_registers = 0x555555578520, 
  tab_registers = 0x5555555784d0}
(gdb) p *mb_mapping
$3 = {nb_bits = 37, start_bits = 304, nb_input_bits = 22, start_input_bits = 452, nb_input_registers = 1, 
  start_input_registers = 264, nb_registers = 32, start_registers = 352, tab_bits = 0x555555578480 "", 
  tab_input_bits = 0x5555555784b0 "", tab_input_registers = 0x555555578520, tab_registers = 0x5555555784d0}
(gdb) p mb_mapping->tab_registers
$4 = (uint16_t *) 0x5555555784d0
(gdb) p *mb_mapping->tab_registers
$5 = 0
(gdb) p *mb_mapping->tab_input_registers 
$6 = 10
(gdb) p mb_mapping->tab_input_registers 
$7 = (uint16_t *) 0x555555578520
(gdb) l
1862	{
1863	    if (mb_mapping == NULL) {
1864	        return;
1865	    }
1866	
1867	    free(mb_mapping->tab_input_registers);
1868	    free(mb_mapping->tab_registers);
1869	    free(mb_mapping->tab_input_bits);
1870	    free(mb_mapping->tab_bits);
1871	    free(mb_mapping);
~~~

### 4. continue to execute

~~~
(gdb) n

Thread 1 "unit-test-serve" hit Breakpoint 2, modbus_mapping_free (mb_mapping=mb_mapping@entry=0x555555578430) at modbus.c:1868
1868	    free(mb_mapping->tab_registers);
(gdb) p mb_mapping->tab_input_registers 
$8 = (uint16_t *) 0x555555578520
(gdb) p *mb_mapping->tab_input_registers 
$9 = 21880
(gdb) n
double free or corruption (out)

Thread 1 "unit-test-serve" received signal SIGABRT, Aborted.
__pthread_kill_implementation (no_tid=0, signo=6, threadid=140737353619264) at ./nptl/pthread_kill.c:44
44	./nptl/pthread_kill.c: No such file or directory.
(gdb) bt
#0  __pthread_kill_implementation (no_tid=0, signo=6, threadid=140737353619264) at ./nptl/pthread_kill.c:44
#1  __pthread_kill_internal (signo=6, threadid=140737353619264) at ./nptl/pthread_kill.c:78
#2  __GI___pthread_kill (threadid=140737353619264, signo=signo@entry=6) at ./nptl/pthread_kill.c:89
#3  0x00007ffff7c42476 in __GI_raise (sig=sig@entry=6) at ../sysdeps/posix/raise.c:26
#4  0x00007ffff7c287f3 in __GI_abort () at ./stdlib/abort.c:79
#5  0x00007ffff7c89676 in __libc_message (action=action@entry=do_abort, fmt=fmt@entry=0x7ffff7ddbb77 "%s\n")
    at ../sysdeps/posix/libc_fatal.c:155
#6  0x00007ffff7ca0cfc in malloc_printerr (str=str@entry=0x7ffff7dde790 "double free or corruption (out)")
    at ./malloc/malloc.c:5664
#7  0x00007ffff7ca2e70 in _int_free (av=0x7ffff7e1ac80 <main_arena>, p=0x5555555784c0, have_lock=<optimized out>)
    at ./malloc/malloc.c:4588
#8  0x00007ffff7ca5453 in __GI___libc_free (mem=<optimized out>) at ./malloc/malloc.c:3391
#9  0x0000555555569022 in modbus_mapping_free (mb_mapping=mb_mapping@entry=0x555555578430) at modbus.c:1868
#10 0x00005555555570bb in main (argc=<optimized out>, argv=<optimized out>) at unit-test-server.c:196
(gdb) 
~~~

As you can see from the gdb debugging flow, on the first execution of free(), no error is reported, when executing the second free(mb_mapping->tab_registers), you can see that the server terminates due to DOUBLE FREE OR CORRUPTION (OUT).