```
layout: default
title: Java 中一个方法如何按照字节码依次执行
```

本文从 Java 虚拟机的角度解释了 Java 中的一个方法如何按照字节码依次执行

1. call_stub

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
看到其中的执行过程，具体过程参见 [call_stub汇编代码注释](https://github.com/codefollower/OpenJDK-Research/blob/master/hotspot/my-docs/interpreter/stub/call_stub.java)
