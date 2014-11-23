---
layout: default
title: Java 中一个方法如何按照字节码依次执行
---

本文从 Java 虚拟机的角度解释了 Java 中的一个方法如何按照字节码依次执行

Java 中的方法调用，最终要落到 `hotspot\src\share\vm\runtime\javaCalls.cpp` 中的 `JavaCalls::call_helper` 函数：

```c++
void JavaCalls::call_helper(JavaValue* result, methodHandle* m, JavaCallArguments* args, TRAPS);
```

其中最为关键的部分在于 

```c++
  // do call
  { JavaCallWrapper link(method, receiver, result, CHECK);
    { HandleMark hm(thread);  // HandleMark used by HandleMarkCleaner

      StubRoutines::call_stub()(
        (address)&link,
        // (intptr_t*)&(result->_value), // see NOTE above (compiler problem)
        result_val_address,          // see NOTE above (compiler problem)
        result_type,
        method(),
        entry_point,
        args->parameters(),
        args->size_of_parameters(),
        CHECK
      );

      result = link.result();  // circumvent MS C++ 5.0 compiler bug (result is clobbered across call)
      // Preserve oop return value across possible gc points
      if (oop_result_flag) {
        thread->set_vm_result((oop) result->get_jobject());
      }
    }
  }
```

call_stub() 返回一个函数指针，括号内的是调用这个函数时传递过去的各种参数，在使用 visual studio 进行调试时，需要进入反汇编才能
看到其中的执行过程，具体过程参见 [call_stub汇编代码注释](https://github.com/codefollower/OpenJDK-Research/blob/master/hotspot/my-docs/interpreter/stub/call_stub.java)。注意汇编中代码注释中栈中参数位置与上述C++代码参数位置的对应关系。在汇编代码中最为关键的是

```asm
02660430  mov         ebx,dword ptr [ebp+14h]  ; 将参数 method() 放入 ebx
02660433  mov         eax,dword ptr [ebp+18h]  ; 将参数 entry_point 放入 eax
02660436  mov         esi,esp  
02660438  call        eax  ; 进入方法调用入口
```

一个 Java 方法调用时，在虚拟机内部并不是直接按字节码顺序执行，在此之前要先通过 call_stub 进入一个入口函数，一般会是 zerolocals， zerolocals是一段汇编代码，在虚拟机启动的时候就已经生成好的，它的执行过程对应的 c++ 代码是 `InterpreterGenerator::generate_normal_entry` ，位于 `hotspot\src\cpu\x86\vm\templateInterpreter_x86_32.cpp`

在虚拟机启动时，通过下面的调用栈：

```
	jvm.dll!TemplateInterpreterGenerator::generate_all() 
 	jvm.dll!InterpreterGenerator::InterpreterGenerator(StubQueue * code)
 	jvm.dll!TemplateInterpreter::initialize()  
 	jvm.dll!interpreter_init() 
 	jvm.dll!init_globals()  
 	jvm.dll!Threads::create_vm(JavaVMInitArgs * args, bool * canTryAgain)  行 3424 + 0x5 字节
 	jvm.dll!JNI_CreateJavaVM(JavaVM_ * * vm, void * * penv, void * args)  行 5166 + 0xd 字节

```

在 `generate_all` 中会生成大量常用代码片断的汇编代码，在执行时直接跳入之前生成好的汇编代码，这里的 `entry_point` 就是zerolocals 对应汇编代码的入口，通过 call 跳转过去。然后，就进入 [zerolocals 的汇编代码](https://github.com/codefollower/OpenJDK-Research/blob/master/hotspot/my-docs/interpreter/stub/method_entry_point_zerolocals.java)，其中最重要的是 

```asm
0266C1A2  mov         esi,dword ptr [ebx+8]  ; ebx + 8 指向 ConstMethod
0266C1A5  lea         esi,[esi+28h]  ; esi 指向方法中第一条字节码
```

注意这里的 ebx 之前已经指向 method，现在通过 ebx+8 指向 ConstMethod，存入 esi，再通过 esi+28h 找到第一条字节码地址，随后就要
跳转到第一条字节码对应的汇编指令入口

```asm
0266AB8D  movzx       ebx,byte ptr [esi]  ; 将第一条字节码内容存入 ebx
0266AB90  jmp         dword ptr [ebx*4+5B436030h]  ; 跳转到第一条字节码对应的汇编指令入口
```

`5B436030h` 是一个表的起始地址，每四个字节代码一个字节码的汇编指令入口，所以通过 `[ebx*4+5B436030h]` 可以找到 ebx 对应的汇编
代码入口。执行完一条字节码内容后，esi 为加1，然后 esi 会指向方法中的第二条字节码，第一条字节码汇编代码的最后，会再次通过 

```asm
0266AB8D  movzx       ebx,byte ptr [esi]  ; 将下一条字节码内容存入 ebx
0266AB90  jmp         dword ptr [ebx*4+5B436030h]  ; 跳转到下一条字节码对应的汇编指令入口
```

跳转到下一条字节码对应的汇编入口。


