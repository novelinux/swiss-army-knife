Debugging Native Code
================================================================================

Capturing logs
To capture log output:

    Produce a process list with ps (ps -t if you want verbose thread feedback).
    Dump kernel messages with dmesg.
    Get verbose log messages with logcat '*:v' & (running in bg with & is important).

Debug Scenarios
--------------------------------------------------------------------------------

  # command to device shell (via adb)
  % command to host pc shell

Crash but no exit...stuck

In this scenario, the GTalk app crashed but did not actually exit or seems stuck. Check the debug logs to see if there is anything unusual:

```
# logcat &
...
E/WindowManager(  182): Window client android.util.BinderProxy@4089f948 has died!!  Removing window.
W/WindowManager(  182): **** WINDOW CLIENT android.view.WindowProxy@40882248 DIED!
W/ActivityManager(  182): **** APPLICATION com.google.android.gtalk DIED!
I/ServiceManager(  257): Executing: /android/bin/app_process (link=/tmp/android-servicemanager/com.google.android.gtalk, wrapper=/tmp/android-servi
cemanager/com.google.android.gtalk)
I/appproc (  257): App process is starting with pid=257, class=android/activity/ActivityThread.
I/        (  257): java.io.FileDescriptor: class initialization
I/SurfaceFlinger.HW(  182): About to give-up screen
I/SurfaceFlinger.HW(  182): screen given-up
I/SurfaceFlinger.HW(  182): Screen about to return
I/SurfaceFlinger.HW(  182): screen returned
I/SurfaceFlinger.HW(  182): About to give-up screen
I/SurfaceFlinger.HW(  182): screen given-up
I/SurfaceFlinger.HW(  182): Screen about to return
...
```

The logs indicate that the system launched a replacement GTalk process but that it got stuck somehow:

```
# ps
PID   PPID  VSIZE RSS   WCHAN    PC         NAME
257   181   45780 5292  ffffffff 53030cb4 S com.google.andr
```

GTalk's PC is at 53030cb4. Look at the memory map to find out what lib is 0x53......

```
# cat /proc/257/maps
...
51000000-5107c000 rwxp 00000000 1f:03 619        /android/lib/libutils.so
52000000-52013000 rwxp 00000000 1f:03 639        /android/lib/libz.so
53000000-53039000 rwxp 00000000 1f:03 668        /android/lib/libc.so
53039000-53042000 rw-p 53039000 00:00 0
54000000-54002000 rwxp 00000000 1f:03 658        /android/lib/libstdc++.so
...
```

Disassemble libc to figure out what is going on:

```
% prebuilt/Linux/toolchain-eabi-4.2.1/bin/arm-elf-objdump -d out/target/product/sooner/symbols/android/lib/libc.so

00030ca4 <__futex_wait>:
  30ca4:       e1a03002        mov     r3, r2
  30ca8:       e1a02001        mov     r2, r1
  30cac:       e3a01000        mov     r1, #0  ; 0x0
  30cb0:       ef9000f0        swi     0x009000f0
  30cb4:       e12fff1e        bx      lr
```

Blocked in a syscall

In this scenario, the system is blocked in a syscall. To debug using gdb, first tell adb to forward the gdb port:

```
% adb forward tcp:5039 tcp:5039
```

Start the gdb server and attach to process 257 (as demonstrated in the previous example):

```
# gdbserver :5039 --attach 257 &
Attached; pid = 257
Listening on port 5039

% prebuilt/Linux/toolchain-eabi-4.2.1/bin/arm-elf-gdb out/target/product/sooner/system/bin/app_process
(gdb) set solib-absolute-prefix /work/android/device/out/target/product/sooner/symbols
(gdb) set solib-search-path /work/android/device/out/target/product/sooner/symbols/android/lib
(gdb) target remote :5039
Remote debugging using :5039
0x53030cb4 in ?? ()
Current language:  auto; currently asm
```

Don't let other threads get scheduled while we're debugging. You should "set scheduler-locking off" before issuing a "continue",
or else your thread may get stuck on a futex or other spinlock because no other thread can release it.

```
(gdb) set scheduler-locking on

Ignore SIGUSR1 if you're using JamVM. Shouldn't hurt if you're not.

(gdb) handle SIGUSR1 noprint

(gdb) where
#0  __futex_wait () at system/klibc/android/atomics_arm.S:88
#1  0x53010eb8 in pthread_cond_timedwait (cond=0x12081c, mutex=0x120818, abstime=0xffffffff)
   at system/klibc/android/pthread.c:490
#2  0x6b01c848 in monitorWait (mon=0x120818, self=0x6b039ba4, ms=0, ns=0) at extlibs/jamvm-1.4.1/src/lock.c:194
#3  0x6b01d1d8 in objectWait (obj=0x408091c0, ms=0, ns=0) at extlibs/jamvm-1.4.1/src/lock.c:420
#4  0x6b01d4c8 in jamWait (clazz=0xfffffffc, mb=0x0, ostack=0x2e188) at extlibs/jamvm-1.4.1/src/natives.c:91
#5  0x6b013b2c in resolveNativeWrapper (clazz=0x408001d0, mb=0x41798, ostack=0x2e188) at extlibs/jamvm-1.4.1/src/dll.c:236
#6  0x6b015c04 in executeJava () at extlibs/jamvm-1.4.1/src/interp.c:2614
#7  0x6b01471c in executeMethodVaList (ob=0x0, clazz=0x40808f20, mb=0x12563c, jargs=0xbe9229f4)
   at extlibs/jamvm-1.4.1/src/execute.c:91
#8  0x6b01bcd0 in Jam_CallStaticVoidMethod (env=0xfffffffc, klass=0x0, methodID=0x12563c)
   at extlibs/jamvm-1.4.1/src/jni.c:1063
#9  0x58025b2c in android::AndroidRuntime::callStatic (this=0xfffffffc,
   className=0xbe922f0a "android/activity/ActivityThread", methodName=0x57000b7c "main")
   at libs/android_runtime/AndroidRuntime.cpp:215
#10 0x57000504 in android::app_init (className=0xbe922f0a "android/activity/ActivityThread")
   at servers/app/library/app_init.cpp:20
#11 0x000089b0 in android::sp<android::ProcessState>::~sp ()
#12 0x000089b0 in android::sp<android::ProcessState>::~sp ()
Previous frame identical to this frame (corrupt stack?)

(gdb) info threads
 7 thread 263  __ioctl () at system/klibc/syscalls/__ioctl.S:12
 6 thread 262  accept () at system/klibc/syscalls/accept.S:12
 5 thread 261  __futex_wait () at system/klibc/android/atomics_arm.S:88
 4 thread 260  __futex_wait () at system/klibc/android/atomics_arm.S:88
 3 thread 259  __futex_wait () at system/klibc/android/atomics_arm.S:88
 2 thread 258  __sigsuspend () at system/klibc/syscalls/__sigsuspend.S:12
 1 thread 257  __futex_wait () at system/klibc/android/atomics_arm.S:88

(gdb) thread 7
[Switching to thread 7 (thread 263)]#0  __ioctl () at system/klibc/syscalls/__ioctl.S:12
12          movs    r0, r0
(gdb) bt
#0  __ioctl () at system/klibc/syscalls/__ioctl.S:12
#1  0x53010704 in ioctl (fd=-512, request=-1072143871) at system/klibc/android/ioctl.c:22
#2  0x51040ac0 in android::IPCThreadState::talkWithDriver (this=0x1207b8, doReceive=true) at RefBase.h:83
#3  0x510418a0 in android::IPCThreadState::joinThreadPool (this=0x1207b8, isMain=false)
   at libs/utils/IPCThreadState.cpp:343
#4  0x51046004 in android::PoolThread::threadLoop (this=0xfffffe00) at libs/utils/ProcessState.cpp:52
#5  0x51036428 in android::Thread::_threadLoop (user=0xfffffe00) at libs/utils/Threads.cpp:1100
#6  0x58025c68 in android::AndroidRuntime::javaThreadShell (args=0x105ffe28) at libs/android_runtime/AndroidRuntime.cpp:540

(gdb) thread 6
[Switching to thread 6 (thread 262)]#0  accept () at system/klibc/syscalls/accept.S:12
12          movs    r0, r0
(gdb) bt
#0  accept () at system/klibc/syscalls/accept.S:12
#1  0x6b0334e4 in jdwpAcceptConnection (state=0xfffffe00) at extlibs/jamvm-1.4.1/jdwp/JdwpNet.c:213
#2  0x6b032660 in jdwpThreadEntry (self=0x4d020) at extlibs/jamvm-1.4.1/jdwp/JdwpMain.c:37
#3  0x6b022c2c in shell (args=0x4d960) at extlibs/jamvm-1.4.1/src/thread.c:629

(gdb) thread 5
[Switching to thread 5 (thread 261)]#0  __futex_wait () at system/klibc/android/atomics_arm.S:88
88              bx              lr
(gdb) bt
#0  __futex_wait () at system/klibc/android/atomics_arm.S:88
#1  0x53010f48 in pthread_cond_timeout (cond=0x6b039b64, mutex=0x6b039b60, msecs=0) at system/klibc/android/pthread.c:513
#2  0x6b01c8d0 in monitorWait (mon=0x6b039b60, self=0x4d400, ms=1000, ns=272629312) at extlibs/jamvm-1.4.1/src/lock.c:183
#3  0x6b022084 in threadSleep (thread=0x4d400, ms=1000, ns=272629312) at extlibs/jamvm-1.4.1/src/thread.c:215
#4  0x6b00d4fc in asyncGCThreadLoop (self=0x4d400) at extlibs/jamvm-1.4.1/src/alloc.c:1179
#5  0x6b022c2c in shell (args=0x4d480) at extlibs/jamvm-1.4.1/src/thread.c:629

(gdb) thread 4
[Switching to thread 4 (thread 260)]#0  __futex_wait () at system/klibc/android/atomics_arm.S:88
88              bx              lr
(gdb) bt
#0  __futex_wait () at system/klibc/android/atomics_arm.S:88
#1  0x53010eb8 in pthread_cond_timedwait (cond=0x6b039934, mutex=0x6b039930, abstime=0x0)
   at system/klibc/android/pthread.c:490
#2  0x6b00b3ec in referenceHandlerThreadLoop (self=0x4d360) at extlibs/jamvm-1.4.1/src/alloc.c:1247
#3  0x6b022c2c in shell (args=0x4d960) at extlibs/jamvm-1.4.1/src/thread.c:629

(gdb) thread 3
[Switching to thread 3 (thread 259)]#0  __futex_wait () at system/klibc/android/atomics_arm.S:88
88              bx              lr
(gdb) bt
#0  __futex_wait () at system/klibc/android/atomics_arm.S:88
#1  0x53010eb8 in pthread_cond_timedwait (cond=0x6b03992c, mutex=0x6b039928, abstime=0x0)
   at system/klibc/android/pthread.c:490
#2  0x6b00b1dc in finalizerThreadLoop (self=0x4d8e0) at extlibs/jamvm-1.4.1/src/alloc.c:1238
#3  0x6b022c2c in shell (args=0x4d960) at extlibs/jamvm-1.4.1/src/thread.c:629

(gdb) thread 2
[Switching to thread 2 (thread 258)]#0  __sigsuspend () at system/klibc/syscalls/__sigsuspend.S:12
12          movs    r0, r0
(gdb) bt
#0  __sigsuspend () at system/klibc/syscalls/__sigsuspend.S:12
#1  0x6b023814 in dumpThreadsLoop (self=0x51b98) at extlibs/jamvm-1.4.1/src/thread.c:1107
#2  0x6b022c2c in shell (args=0x51b58) at extlibs/jamvm-1.4.1/src/thread.c:629
```

Crash in C / C++ code
--------------------------------------------------------------------------------

If it crashes, connect with aproto and run logcat on the device. You should see output like this:

```
I/ActivityManager(  188): Starting activity: Intent { component=com.android.calendar.MonthScreen }
I/ActivityManager(  188): Starting application com.android.calendar to host activity com.android.calendar.MonthScree
n
I/ServiceManager(  417): Executing: /android/bin/app_process (link=/android/bin/app_process, wrapper=/android/bin/app_process)
I/DEBUG: -- observer of pid 417 starting --
I/appproc (  417): App process is starting with pid=417, class=android/activity/ActivityThread.
I/DEBUG: -- observer of pid 417 exiting --
I/DEBUG: -- observer of pid 420 starting --
I/DEBUG: *** *** *** *** *** *** *** *** *** *** *** *** *** *** *** ***
I/DEBUG: pid: 373, tid: 401  >>> android.content.providers.pim <<<
I/DEBUG: signal 11 (SIGSEGV), fault addr 00000000
I/DEBUG:  r0 ffffffff  r1 00000000  r2 00000454  r3 002136d4
I/DEBUG:  r4 002136c0  r5 40804810  r6 0022dc70  r7 00000010
I/DEBUG:  r8 0020a258  r9 00000014  10 6b039074  fp 109ffcf8
I/DEBUG:  ip 6b039e90  sp 109ffc0c  lr 580239f0  pc 6b0156a0
I/DEBUG:          #01  pc 6b0156a0  /android/lib/libjamvm.so
I/DEBUG:          #01  lr 580239f0  /android/lib/libandroid_runtime.so
I/DEBUG:          #02  pc 6b01481c  /android/lib/libjamvm.so
I/DEBUG:          #03  pc 6b0148a4  /android/lib/libjamvm.so
I/DEBUG:          #04  pc 6b00ebc0  /android/lib/libjamvm.so
I/DEBUG:          #05  pc 6b02166c  /android/lib/libjamvm.so
I/DEBUG:          #06  pc 6b01657c  /android/lib/libjamvm.so
I/DEBUG:          #07  pc 6b01481c  /android/lib/libjamvm.so
I/DEBUG:          #08  pc 6b0148a4  /android/lib/libjamvm.so
I/DEBUG:          #09  pc 6b0235c0  /android/lib/libjamvm.so
I/DEBUG:          #10  pc 5300fac4  /android/lib/libc.so
I/DEBUG:          #11  pc 5300fc5c  /android/lib/libc.so
I/DEBUG: -- observer of pid 373 exiting --
I/DEBUG: -- observer of pid 423 starting --
```

If debugging output indicates an error in C or C++ code, the addresses aren't particularly useful,
but the debugging symbols aren't present on the device. Use the "stack" tool to convert these addresses to files and line numbers,
for example:

```
pid: 373, tid: 401  >>> android.content.providers.pim <<<

 signal 11 (SIGSEGV), fault addr 00000000
  r0 ffffffff  r1 00000000  r2 00000454  r3 002136d4
  r4 002136c0  r5 40804810  r6 0022dc70  r7 00000010
  r8 0020a258  r9 00000014  10 6b039074  fp 109ffcf8
  r8 0020a258  r9 00000014  10 6b039074  fp 109ffcf8

  ADDR      FUNCTION                        FILE:LINE
  6b0156a0  executeJava                     extlibs/jamvm-1.4.1/src/interp.c:2674
  580239f0  android_util_Parcel_freeBuffer  libs/android_runtime/android_util_Binder.cpp:765
  6b01481c  executeMethodVaList             extlibs/jamvm- 1.4.1/src/execute.c:91
  6b0148a4  executeMethodArgs               extlibs/jamvm-1.4.1/src/execute.c:67
  6b00ebc0  initClass                       extlibs/jamvm-1.4.1/src/class.c:1124
  6b02166c  resolveMethod                   extlibs/jamvm- 1.4.1/src/resolve.c:197
  6b01657c  executeJava                     extlibs/jamvm-1.4.1/src/interp.c:2237
  6b01481c  executeMethodVaList             extlibs/jamvm-1.4.1/src/execute.c:91
  6b0148a4  executeMethodArgs               extlibs/jamvm- 1.4.1/src/execute.c:67
  6b0235c0  threadStart                     extlibs/jamvm-1.4.1/src/thread.c:355
  5300fac4  __thread_entry                  system/klibc/android/pthread.c:59
  5300fc5c  pthread_create                  system/klibc/android/pthread.c:182
```

Or you can run logcat without any parameters and it will read from stdin. You can then paste output into
the terminal or pipe it. Run logcat from the top of the tree in the environment in which you do builds
so that the application can determine relative paths to the toolchain to use to decode the object files.
