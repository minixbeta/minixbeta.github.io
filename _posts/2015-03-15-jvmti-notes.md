---
layout: post
title: JVMTI 中需要注意的几个问题
comments: true
category: 技术
---

## jni functions
在使用 JVMTI 的过程中，有一大系列的函数是在 [JVMTI 的文档](http://docs.oracle.com/javase/7/docs/platform/jvmti/jvmti.html)中
没有提及的，但在实际使用却是非常有用的。这就是 [jni functions](http://docs.oracle.com/javase/7/docs/technotes/guides/jni/spec/functions.html).

例如，在使用 SingleStep 函数时，

```
void JNICALL
SingleStep(jvmtiEnv *jvmti_env,
            JNIEnv* jni_env,
            jthread thread,
            jmethodID method,
            jlocation location)
```

我们就可以使用 jni_env 来调用 jni functions 中的一系列函数。例如：

```
jclass mainClass = (*jni_env)->FindClass(jni_env, "Foo");
jfieldID fieldID = (*jni_env)->GetStaticFieldID(jni_env, mainClass, "x", "I");
jint fieldValue = (*jni_env)->GetStaticIntField(jni_env, mainClass, fieldID);
```

通过使用 jni functions 就可以获取 Foo 类中的静态整型变量的值。

另外在 [jni functions 文档](http://docs.oracle.com/javase/7/docs/technotes/guides/jni/spec/functions.html) 中还有大量易用的
函数，都是在 JVMTI 文档中没有提及的，非常有用。

## 事件回调函数的线程安全
在 JVMTI 中有一系列回调函数，当使用 `SetEventNotificationMode` 和 `SetEventCallbacks` 设置了某个事件的回调函数后，Java 虚拟机
在相应事件发生时，就会调用我们在 JVMTI 的 agent 中写的回调函数。

千万注意：`多数回调函数需要保证线程安全`。

也就是说，我们写的回调函数要保证是可重入的，不然的话，一个线程执行时进入了回调函数，还没执行完这个回调函数时，可能会被切换
到另一线程，而另一线程也可能重新进入这个回调函数，执行完之后，又切换回前一线程时，才继续执行上一个没有执行完的回调函数。如果
你使用了全局变量，就要格外注意上述情况可能导致的数据不一致问题。

解决这个问题的最简单的办法就是使用 Raw Monitor。

在 Agent_OnLoad 函数中创建一个 Raw Monitor：

```
(*jvmti)->CreateRawMonitor(jvmti, "singelStep", &singleStepMonitorId);
```

然后，在回调函数的开始和结束时，使用这个 Raw Monitor 形成一个临界区，使得多个线程不能同时进入这个临界区:

```
void JNICALL
callbackSingleStep(
jvmtiEnv *jvmti,
JNIEnv* jni,
jthread thread,
jmethodID method,
jlocation location) {

(*jvmti)->RawMonitorEnter(jvmti, singleStepMonitorId);

...

(*jvmti)->RawMonitorExit(jvmti, singleStepMonitorId);
}
```

## JVMTI agent 的调试
JVMTI 的 agent 如果出错，Java 虚拟机就会直接崩溃，同时生成一个 log，我们需要根据这个 log 猜测 agent 中哪里出现了错误，例如：

```
#
# A fatal error has been detected by the Java Runtime Environment:
#
#  EXCEPTION_ACCESS_VIOLATION (0xc0000005) at pc=0x000000006f99f004, pid=6760, tid=8140
#
# JRE version: Java(TM) SE Runtime Environment (8.0_31-b13) (build 1.8.0_31-b13)
# Java VM: Java HotSpot(TM) 64-Bit Server VM (25.31-b07 mixed mode windows-amd64 compressed oops)
# Problematic frame:
# V  [jvm.dll+0x12f004]
#
# Failed to write core dump. Minidumps are not enabled by default on client versions of Windows
#
# If you would like to submit a bug report, please visit:
#   http://bugreport.java.com/bugreport/crash.jsp
#

...

Register to memory mapping:

RAX=0x0000000000000000 is an unknown value
RBX=0x000000005af15000 is a thread
RCX=0x000000005af15000 is a thread
RDX=0x00000000d707b160 is an oop
java.lang.NoSuchFieldError 
 - klass: 'java/lang/NoSuchFieldError'
RSP=0x000000005bcce510 is pointing into the stack for thread: 0x000000005af15000
RBP=0x000000005bcceb80 is pointing into the stack for thread: 0x000000005af15000
RSI={method} {0x0000000056b83b18} 'run' '()V' in 'TestCaset1000$1'
RDI=0x0000000000000000 is an unknown value
R8 =0x0000000000000000 is an unknown value
R9 =0x0000000000000b80 is an unknown value
R10=0xfefefefefefefeff is an unknown value
R11=0x8080808080808080 is an unknown value
R12=0x000000005af15000 is a thread
R13=0x000000005ae58030 is an unknown value
R14=0x0000000000000000 is an unknown value
R15=0x00000000024388e0 is an unknown value


Stack: [0x000000005bbd0000,0x000000005bcd0000],  sp=0x000000005bcce510,  free space=1017k
Native frames: (J=compiled Java code, j=interpreted, Vv=VM code, C=native code)
V  [jvm.dll+0x12f004]
C  [TraceCapture.dll+0x5086]  callbackSingleStep+0x466
V  [jvm.dll+0x1a7389]
V  [jvm.dll+0x1aa34c]
V  [jvm.dll+0xa9c23]
C  0x00000000025afddd

...

```
其中 TraceCapture.dll 就是我写的 JVMTI agent，大概可以猜测到是  `callbackSingleStep+0x466` 这个地方出一错误，异常
是 `java.lang.NoSuchFieldError`。那么怎么找到这个位置呢？

这就涉及调试动态链接库文件了，调试的时候，在 callbackSingleStep 函数上加一个断点，调试时会停在这个断点上。
此时，首先查看 callbackSingleStep 的首地址，在 visual studio 2013 中把鼠标放在函数名上就可以显示出来，然后将这个地址增加
0x466，就找到了出错的位置的地址，再在 Call Stack 窗口中，右击 TraceCapture.dll 这一项进行反汇编，就进入了反汇编界面，
找到相应地址对应的函数语句即可。
