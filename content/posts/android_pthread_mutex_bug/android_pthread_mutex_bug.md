+++
title = "A bug related with Android pthread mutex"
date = "2016-09-14T14:00:00"
tags = ["android", "pthread"]
categories = ["work"]
+++

## 概述

这几天在调试一个android系统启动的问题, 现象是系统在启动过程中会hang住,
最终发现根本原因是因为等待在一个mutex上, 这里将分析过程记录下来.

## 分析过程

由于android系统中的关键服务hang住, 系统watchdog会将所有进程的stack dump出来,
( 结果保存在/data/anr/traces.txt )所以首先分析这些stack, 查看系统哪个服务hang住.

### hanged service

首先根据文件开头, 可以确定是system_server服务hang住了. 查看其主线程的stack:
查看其主线程的stack:

```
DALVIK THREADS (69):
"main" prio=5 tid=1 Blocked
  | group="main" sCount=1 dsCount=0 obj=0x751fcfa8 self=0x7f4488095000
  | sysTid=333 nice=-2 cgrp=lxc/1 sched=0/0 handle=0x7f448beb2500
  | state=S schedstat=( 342112604 33453570 1186 ) utm=26 stm=7 core=0 HZ=100
  | stack=0x7ffd3bbe8000-0x7ffd3bbea000 stackSize=8MB
  | held mutexes=
  at com.android.server.am.ActivityManagerService.broadcastIntent(ActivityManagerService.java:16564)
  - waiting to lock <0x3491e366> (a com.android.server.am.ActivityManagerService) held by thread 12
  at android.app.ContextImpl.sendBroadcastAsUser(ContextImpl.java:1443)
  at com.android.server.DropBoxManagerService$3.handleMessage(DropBoxManagerService.java:162)
  at android.os.Handler.dispatchMessage(Handler.java:102)
  at android.os.Looper.loop(Looper.java:135)
  at com.android.server.SystemServer.run(SystemServer.java:269)
  at com.android.server.SystemServer.main(SystemServer.java:170)
  at java.lang.reflect.Method.invoke!(Native method)
  at java.lang.reflect.Method.invoke(Method.java:372)
  at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:903)
  at.code traces.txt 56,74 com.android.internal.os.ZygoteInit.main(ZygoteInit.java:698)
```

看来它在等待一个lock, 而它被thread 12拿住, 再来看thread 12的stack:

```
"Binder_1" prio=5 tid=12 Blocked
  | group="main" sCount=1 dsCount=0 obj=0x12cc60a0 self=0x7f448103b800
  | sysTid=349 nice=0 cgrp=lxc/1 sched=0/0 handle=0x7f4488115900
  | state=S schedstat=( 9129822 12410930 350 ) utm=0 stm=0 core=0 HZ=100
  | stack=0x7f4486b02000-0x7f4486b04000 stackSize=1012KB
  | held mutexes=
  at com.android.server.wm.WindowManagerService.executeAppTransition(WindowManagerService.java:4169)
  - waiting to lock <0x0e765253> (a java.util.HashMap) held by thread 63
  at com.android.server.am.ActivityStack.resumeTopActivityInnerLocked(ActivityStack.java:1509)
  at com.android.server.am.ActivityStack.resumeTopActivityLocked(ActivityStack.java:1455)
  at com.android.server.am.ActivityStackSupervisor.resumeTopActivitiesLocked(ActivityStackSupervisor.java:2525)
  at com.android.server.am.ActivityStackSupervisor.resumeTopActivitiesLocked(ActivityStackSupervisor.java:2514)
  at com.android.server.am.ActivityManagerService.handleAppDiedLocked(ActivityManagerService.java:4797)
  at com.android.server.am.ActivityManagerService.appDiedLocked(ActivityManagerService.java:4936)
  at com.android.server.am.ActivityManagerService$AppDeathRecipient.binderDied(ActivityManagerService.java:1276)
  - locked <0x3491e366> (a com.android.server.am.ActivityManagerService)
  at android.os.BinderProxy.sendDeathNotice(Binder.java:551)
```

同样等待一个lock, 它由thread 63那住 , 在看看thread 63的stack :

```
"Binder_5" prio=5 tid=63 Native
  | group="main" sCount=1 dsCount=0 obj=0x131190a0 self=0x7f447e1cd000
  | sysTid=617 nice=0 cgrp=lxc/1 sched=0/0 handle=0x7f447e218a00
  | state=S schedstat=( 5362354 6320934 301 ) utm=0 stm=0 core=1 HZ=100
  | stack=0x7f44716d8000-0x7f44716da000 stackSize=1012KB
  | held mutexes=
  kernel: timerqueue_del+0x4c/0x53
  kernel: binder_thread_read+0x3c4/0xcfb
  kernel: hrtimer_cancel+0x11/0x1b
  kernel: autoremove_wake_function+0x0/0x2f
  kernel: binder_ioctl_write_read+0x121/0x234
  kernel: binder_ioctl+0x158/0x4a9
  kernel: do_vfs_ioctl+0x36c/0x430
  kernel: __fget+0x69/0x71
  kernel: SyS_ioctl+0x4e/0x7c
  kernel: stub_rt_sigreturn+0x6d/0xa0
  kernel: system_call_fastpath+0x16/0x1b
  native: #00 pc 00079207  /system/lib64/libc.so (__ioctl+7)
  native: #01 pc 000aebc9  /system/lib64/libc.so (ioctl+41)
  native: #02 pc 000304fe  /system/lib64/libbinder.so (android::IPCThreadState::talkWithDriver(bool)+190)
  native: #03 pc 00030fa8  /system/lib64/libbinder.so (android::IPCThreadState::waitForResponse(android::Parcel*, int*)+120)
  native: #04 pc 0003120f  /system/lib64/libbinder.so (android::IPCThreadState::transact(int, unsigned int, android::Parcel const&, android::Parcel*, unsigned int)+159)
  native: #05 pc 00028ea7  /system/lib64/libbinder.so (android::BpBinder::transact(unsigned int, android::Parcel const&, android::Parcel*, unsigned int)+71)
  native: #06 pc 00057bba  /system/lib64/libgui.so (android::BpSurfaceComposerClient::createSurface(android::String8 const&, unsigned int, unsigned int, int, unsigned int, android::sp<android::IBinder>*, android::sp<android::IGraphicBufferProducer>*)+202)
  native: #07 pc 00062ce8  /system/lib64/libgui.so (android::SurfaceComposerClient::createSurface(android::String8 const&, unsigned int, unsigned int, int, unsigned int)+104)
  native: #08 pc 000db4ff  /system/lib64/libandroid_runtime.so (???)
  native: #09 pc 013ea29b  /data/dalvik-cache/x86_64/system@framework@boot.oat (Java_android_view_SurfaceControl_nativeCreate__Landroid_view_SurfaceSession_2Ljava_lang_String_2IIII+287)
  at android.view.SurfaceControl.nativeCreate(Native method)
  at android.view.SurfaceControl.<init>(SurfaceControl.java:287)
  at com.android.server.wm.WindowStateAnimator.createSurfaceLocked(WindowStateAnimator.java:861)
  at com.android.server.wm.WindowManagerService.relayoutWindow(WindowManagerService.java:3172)
  - locked <0x0e765253> (a java.util.HashMap)
  at com.android.server.wm.Session.relayout(Session.java:197)
  at android.view.IWindowSession$Stub.onTransact(IWindowSession.java:273)
  at com.android.server.wm.Session.onTransact(Session.java:130)
  at android.os.Binder.execTransact(Binder.java:446)
```

可以看出 , 这里thread 63在等待一个binder的回应 , 通过stack可以看出该binder
是要创建一个surface , 在android系统中surface的创建都是统一由surfaceflinger
服务创建的，所以该binder是在等待surfaceflinger创建surface的结果 .
那么我们来看看surfaceflinger服务这时在做什么 .

首先来看看surfaceflinger进程中处理该binder请求的thread的stack :

```
"Binder_3" sysTid=651
  #00 pc 000000000001a9b8  /system/lib64/libc.so (syscall+24)
  #01 pc 0000000000028302  /system/lib64/libc.so (pthread_cond_wait+82)
  #02 pc 000000000002c84a  /system/lib64/libsurfaceflinger.so
  #03 pc 000000000001bc6c  /system/lib64/libsurfaceflinger.so
  #04 pc 0000000000057e03  /system/lib64/libgui.so (android::BnSurfaceComposerClient::onTransact(unsigned int, android::Parcel const&, android::Parcel*, unsigned int)+307)
  #05 pc 0000000000027fe0  /system/lib64/libbinder.so (android::BBinder::transact(unsigned int, android::Parcel const&, android::Parcel*, unsigned int)+208)
  #06 pc 0000000000030b9b  /system/lib64/libbinder.so (android::IPCThreadState::executeCommand(int)+875)
  #07 pc 0000000000030de1  /system/lib64/libbinder.so (android::IPCThreadState::getAndExecuteCommand()+65)
  #08 pc 0000000000030e47  /system/lib64/libbinder.so (android::IPCThreadState::joinThreadPool(bool)+71)
  #09 pc 00000000000380a6  /system/lib64/libbinder.so (android::PoolThread::threadLoop()+22)
  #10 pc 00000000000185f7  /system/lib64/libutils.so (android::Thread::_threadLoop(void*)+295)
  #11 pc 00000000000284ce  /system/lib64/libc.so (__pthread_start(void*)+46)
  #12 pc 000000000002430b  /system/lib64/libc.so (__start_thread+11)
  #13 pc 000000000001a7e5  /system/lib64/libc.so (__bionic_clone+53)
```

这里是在等待一个信号量 , 这个信号量又是做什么的呢 ?
是这样的，binder处理thread收到一个请求后 ( 这里就是创建surface的请求 )
会将该请求转发给surfaceflinger主线程去处理 , 当主线程处理结束 ,
会将处理结果返回给binder处理线程 , 而binder处理线程再将结果转发给请求者 .
所以，这里其实是在等待主线程的处理结果 . 那么我们来看看主线程的stack :

```
"surfaceflinger" sysTid=54
  #00 pc 0000000000010af8  /system/bin/linker64 (__dl_syscall+24)
  #01 pc 000000000000f6c2  /system/bin/linker64 (__dl_pthread_mutex_lock+290)
  #02 pc 0000000000002186  /system/bin/linker64 (__dl_dlopen+22)
  #03 pc 00000000000009f4  /system/lib64/libhardware.so (hw_get_module_by_class+228)
  #04 pc 000000000001a37f  /system/lib64/egl/libGLES_android.so
  #05 pc 0000000000007a57  /system/lib64/egl/libGLES_android.so (glDrawArrays+1271)
  #06 pc 00000000000441af  /system/lib64/libsurfaceflinger.so
  #07 pc 0000000000026a57  /system/lib64/libsurfaceflinger.so
  #08 pc 000000000002667e  /system/lib64/libsurfaceflinger.so
  #09 pc 000000000002ed66  /system/lib64/libsurfaceflinger.so
  #10 pc 000000000002faca  /system/lib64/libsurfaceflinger.so
  #11 pc 000000000002e4e1  /system/lib64/libsurfaceflinger.so
  #12 pc 000000000002d215  /system/lib64/libsurfaceflinger.so
  #13 pc 000000000002cf08  /system/lib64/libsurfaceflinger.so
  #14 pc 000000000001c80e  /system/lib64/libutils.so (android::Looper::pollInner(int)+334)
  #15 pc 000000000001cb2d  /system/lib64/libutils.so (android::Looper::pollOnce(int, int*, int*, void**)+61)
  #16 pc 00000000000294a1  /system/lib64/libsurfaceflinger.so
  #17 pc 000000000002c9b7  /system/lib64/libsurfaceflinger.so (android::SurfaceFlinger::run()+23)
  #18 pc 0000000000001066  /system/bin/surfaceflinger
  #19 pc 000000000001a6bc  /system/lib64/libc.so (__libc_init+92)
  #20 pc 0000000000001151  /system/bin/surfaceflinger
  #21 pc 0000000000000000  <unknown>
```

原来主线程在做合成操作时等待一个mutex , 那么这个mutex被谁拿着呢 ?
另外 , 通过查看surfaceflinger的其他线程 , 发现其中这个thread很奇怪 ,
它也在等待同一个mutex :

```
"ConsoleManagerT" sysTid=64
  #00 pc 0000000000010af8  /system/bin/linker64 (__dl_syscall+24)
  #01 pc 000000000000f6c2  /system/bin/linker64 (__dl_pthread_mutex_lock+290)
  #02 pc 0000000000002237  /system/bin/linker64 (__dl_dlsym+23)
  #03 pc 00000000000016c5  /system/bin/surfaceflinger (InitializeSignalChain+69)
  #04 pc 000000000000137a  /system/bin/surfaceflinger (sigprocmask+154)
  #05 pc 0000000000028a93  /system/lib64/libc.so (pthread_exit+227)
  #06 pc 00000000000284d6  /system/lib64/libc.so (__pthread_start(void*)+54)
  #07 pc 000000000002430b  /system/lib64/libc.so (__start_thread+11)
  #08 pc 000000000001a7e5  /system/lib64/libc.so (__bionic_clone+53)
```

### pthread mutex

通过查看相关source , 该mutex是linker中的全局mutex , 当我们调用dynamic
linker的相关接口 ( dlopen , dlclose , dlsym 等 ) 就需要acquire 该 mutex .

```
static void* dlopen_ext(const char* filename, int flags, const android_dlextinfo* extinfo) {
  ScopedPthreadMutexLocker locker(&g_dl_mutex);

void* dlsym(void* handle, const char* symbol) {
  ScopedPthreadMutexLocker locker(&g_dl_mutex);

int dlclose(void* handle) {
  ScopedPthreadMutexLocker locker(&g_dl_mutex);
```

而且发现该锁在使用时 , 为了防止忘记释放的问题，它使用了c++的scope作用域 .
所以该锁在函数返回时都会被释放 .

再来看该mutex的定义 : 

```
static pthread_mutex_t g_dl_mutex = PTHREAD_RECURSIVE_MUTEX_INITIALIZER;
```

需要注意的是该mutex是可以嵌套的 , 也就是说如果该mutex被某个thread拿着 ,
该thread可以再次acquire , 而不会产生dead lock.
那么这里的dead lock又是如何产生的呢 ?

没办法 , 只能从pthread mutex的代码上寻找线索 . 
经过阅读bionic的pthread mutex实现源码 , 在mutex的unlock
函数中有一点比较怀疑 : 

```
/* Do we already own this recursive or error-check mutex ? */
tid = __get_thread()->tid;
if ( tid != MUTEX_OWNER_FROM_BITS(mvalue) ) {
    return EPERM;
}
```

这里的意思是在释放锁之前会检查调用者是否是该锁的持有者 ,
如果不是的话返回permission deny . 那么会不会是这里的问题呢 ,
通过在这里添加log , 发现的确是这里的问题 .

```
/* Do we already own this recursive or error-check mutex ? */
tid = __get_thread()->tid;
if ( tid != MUTEX_OWNER_FROM_BITS(mvalue) ) {
+	__libc_format_log(ANDROID_LOG_FATAL, "libc", "caller[%d] != holder[%d]\n", tid, MUTEX_OWNER_FROM_BITS(mvalue));
	return EPERM;
}

log:
...
F/libc    ( 1116): caller[0] != holder[1123]
...
```

可以看出这里调用者的tid为0 ??? 而该mutex的持有者的tid为1123.
从而导致该mutex释放失败 , 所以之后尝试拿该mutex都会被阻塞住 .

所以 , 下面的问题就是弄清为什么尝试释放mutex的线程的tid和当前mutex
持有者的tid不同 , 而且其tid明显是一个非法的值 ( 这里指0 ) .

### thread tid

首先我们来看tid是从哪里获得的:

```
tid = __get_thread()->tid;
```

而 `__get_thread()` : 

```
pthread_internal_t* __get_thread(void) {
  return reinterpret_cast<pthread_internal_t*>(__get_tls()[TLS_SLOT_THREAD_ID]);
}
```

原来tid是从保存在 `tls` 中的 `pthread_internal_t` 的tid成员中.
而 `pthread_internal_t` 初始化工作是在 `pthread_create` 最开始完成的 : 

```
int pthread_create(pthread_t* thread_out, pthread_attr_t const* attr,
                   void* (*start_routine)(void*), void* arg) {
	...

  pthread_internal_t* thread = reinterpret_cast<pthread_internal_t*>(calloc(sizeof(*thread), 1));
  if (thread == NULL) {
    __libc_format_log(ANDROID_LOG_WARN, "libc", "pthread_create failed: couldn't allocate thread");
    return EAGAIN;
  }

  ...

  __init_tls(thread);
```

```
void __init_tls(pthread_internal_t* thread) {
	...
  thread->tls[TLS_SLOT_THREAD_ID] = thread;
	...
}
```

当调用 `clone` 系统调用之后 , 系统会将创建出来的线程id写入到 `thread->tid` 中 : 

```
int rc = clone(__pthread_start, child_stack, flags, thread, &(thread->tid), tls, &(thread->tid));
```

弄清楚了tid从哪里来之后 , 那我们再来看看 `pthread_internal_t` 的释放.
`pthread_internal_t` 的释放是在 `pthread_exit` 中 :

```
void pthread_exit(void* return_value) {
  pthread_internal_t* thread = __get_thread();

  ...

  if ((thread->attr.flags & PTHREAD_ATTR_FLAG_DETACHED) != 0) {
    // The thread is detached, so we can free the pthread_internal_t.
    // First make sure that the kernel does not try to clear the tid field
    // because we'll have freed the memory before the thread actually exits.
    __set_tid_address(NULL);
    _pthread_internal_remove_locked(thread);
```

```
void _pthread_internal_remove_locked(pthread_internal_t* thread) {
	...
    free(thread);
}
```

注意 , 这里只有被标记detached的thread才会被释放 ,
这类thread的退出不需要通知其父线程 ( 也就是创建该线程的线程 ) .
一般的线程退出之后都会通知其父线程 , 父线程一般通过 `pthread_join` 等待通知 , 
随后释放该线程的相关资源 . 

回忆之前我们提到过一个奇怪的线程也在等待mutex : 

```
"ConsoleManagerT" sysTid=64
  #00 pc 0000000000010af8  /system/bin/linker64 (__dl_syscall+24)
  #01 pc 000000000000f6c2  /system/bin/linker64 (__dl_pthread_mutex_lock+290)
  #02 pc 0000000000002237  /system/bin/linker64 (__dl_dlsym+23)
  #03 pc 00000000000016c5  /system/bin/surfaceflinger (InitializeSignalChain+69)
  #04 pc 000000000000137a  /system/bin/surfaceflinger (sigprocmask+154)
  #05 pc 0000000000028a93  /system/lib64/libc.so (pthread_exit+227)
  #06 pc 00000000000284d6  /system/lib64/libc.so (__pthread_start(void*)+54)
  #07 pc 000000000002430b  /system/lib64/libc.so (__start_thread+11)
  #08 pc 000000000001a7e5  /system/lib64/libc.so (__bionic_clone+53)
```

可以看出它是在 `pthread_exit` 的过程中 ,
通过仔细分析其stack , 可以看出它其实是在执行 `pthread_exit` 最后的 `sigprocmask`
: 

```
// We don't want to take a signal after we've unmapped the stack.
// That's one last thing we can handle in C.
sigset_t mask;
sigfillset(&mask);
sigprocmask(SIG_SETMASK, &mask, NULL);

_exit_with_stack_teardown(stack_base, stack_size);
```

注意 , 这里 `pthread_internal_t` 已经被释放了, 所以这里获取的tid是非法的 , 
也就是 `use` `after` `freed` .
那么为什么没有导致segment fault呢 ? 这里和 `bionic` 的 `malloc/free` 实现有关 ,
bionic的内存管理使用的是jemalloc , 当一块memory被释放, 在一段时间内其仍然是可以
被访问的 , 只不过其内容是未知的 , 所以我们之前会看到tid为0这样的非法值 . 

为了验证这点 , 我写了一个简单的测试程序 :

```
#include <pthread.h>
#include <stdio.h>

static void*
work(void *arg)
{
	pthread_exit(NULL);
}

int
main(int argc, const char *argv[])
{
	pthread_t t;
	pthread_attr_t attr;
	pthread_attr_init(&attr);
	pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);
	pthread_create( &t, &attr, &work, NULL);

	while (1) {
		sleep(1);
	}

	printf("well done\n");
	return 0;
}
```

使用gdb在释放 `pthread_internal_t` 之前将其tid改成一个特殊的值 : 

```
Breakpoint 3, _pthread_internal_remove_locked (thread=0x7ffff741c300) at bionic/libc/bionic/pthread_internals.cpp:39
(gdb) set thread->tid=999
(gdb) p *thread
$1 = {next = 0x7ffff7ffd500 <__dl__ZZ15__libc_init_tlsR19KernelArgumentBlockE11main_thread>, prev = 0x0, tid = 999, cached_pid_ = 12774, tls = 0x7ffff7e65b60, attr = {flags = 1, 
    stack_base = 0x7ffff7d68000, stack_size = 1040384, guard_size = 4096, sched_policy = 0, __sched_priority = 0, __reserved = "\001\000\000\000\000\000\000\000\060IUUUU\000"}, cleanup_stack = 0x0, 
  start_routine = 0x555555554a60 <work2(void*)>, start_routine_arg = 0x0, return_value = 0x0, alternate_signal_stack = 0x0, startup_handshake_mutex = {value = 1, 
    __reserved = '\000' <repeats 35 times>}, dlerror_buffer = '\000' <repeats 511 times>}
```

这里我们将其tid该写成999

之后我们在 `sigprocmask` 中查看已经释放的 `pthread_internal_t` : 

```
Breakpoint 2, art::sigprocmask (how=2, bionic_new_set=0x7ffff7e65ad0, bionic_old_set=0x0) at art/sigchainlib/sigchain.cc:224
224     extern "C" int sigprocmask(int how, const sigset_t* bionic_new_set, sigset_t* bionic_old_set) {
(gdb) p *__get_thread()
$5 = {next = 0x7ffff7ffd500 <__dl__ZZ15__libc_init_tlsR19KernelArgumentBlockE11main_thread>, prev = 0x0, tid = 999, cached_pid_ = 12774, tls = 0x7ffff7e65b60, attr = {flags = 1, 
    stack_base = 0x7ffff7d68000, stack_size = 1040384, guard_size = 4096, sched_policy = 0, __sched_priority = 0, __reserved = "\001\000\000\000\000\000\000\000\060IUUUU\000"}, cleanup_stack = 0x0, 
  start_routine = 0x555555554a60 <work2(void*)>, start_routine_arg = 0x0, return_value = 0x0, alternate_signal_stack = 0x0, startup_handshake_mutex = {value = 1, 
    __reserved = '\000' <repeats 35 times>}, dlerror_buffer = '\000' <repeats 511 times>}
```

和我们预期的一样 , 被释放的 `pthread_internal_t` 仍然可以访问 ,
而且其内容还没有变!!!

## 结尾

至此 , 问题已分析清楚 , 可能的解决方法 , 可以修改 `sigprocmask` 的实现 . 
当然我也给android官方提了个[bug](https://code.google.com/p/android/issues/detail?id=222398).

FIN.
