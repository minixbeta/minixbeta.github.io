---
layout: post
title: JDK8 中的类型推断与重载解析
category: 技术
comments: true
---

首先从一个例子开始：

下面这段代码，在 JDK6u30 中可以正常工作，但是在 JDK8u65 中会运行失败，提示类型转换错误，ClassCastException。

```
Exception in thread "main" java.lang.ClassCastException: java.lang.String cannot be cast to [C 
```


我看了一下字节码，泛型函数推出的返回值都是 Object。然而 JDK8 中调用重载过的函数时，选择了 `String valueOf(char data[])`，本来应该选择 `String.valueOf(Object obj)`，JDK6就是这么做的，为什么到 JDK8 反而选择了一个错误的函数呢？ 

```java
public class TestTypeInference {
    public static <T> T get() {
        return (T) "x";
    }

    public static void main(String[] args) {
        System.out.println(String.valueOf(get()));
    }

// JDK6: 
// INVOKESTATIC TestTypeInference.get ()Ljava/lang/Object;
// INVOKESTATIC java/lang/String.valueOf (Ljava/lang/Object;)Ljava/lang/String;

// JDK8:  
// INVOKESTATIC TestTypeInference.get ()Ljava/lang/Object;
// CHECKCAST [C
// INVOKESTATIC java/lang/String.valueOf ([C)Ljava/lang/String;
}
```

这个问题困扰了我很久，也没有人可以回答我，只好自己去 `javac` 的源代码里找答案。

首先看 JDK6 中如何通过调试 `javac` 编译上面的代码。从 OpenJDK 上把 JDK6 的 [源代码](http://download.java.net/openjdk/jdk6/) 下下来，找到 javac 源代码的入口 `openjdk-6-src-b30-21_jan_2014\langtools\src\share\classes\com\sun\tools\javac\main\Main.java`，一层一层调试下去。发现关键的地方有两个：

  * 对 `get()` 返回类型的推断
  * 对 `String.valueOf` 这个重载方法的选择
  
在 JDK6 中，直接推断出 `get()` 的返回类型是 `Object`，然后对 `String.valueOf` 的所有方法进行遍历，找到可以将 `Object` 作为参数的那个方法。最终选择了 `String.valueOf(Object)`。

推断 `get()` 返回值类型的入口位于 `com.sun.tools.javac.comp.Attr#visitApply` 方法中：

```java
argtypes = attribArgs(tree.args, localEnv);
```

其中 `tree.args` 此时就是 `get()` 方法。

而最终将 `Object` 作为 `get()` 的返回类型，而位于 `com.sun.tools.javac.comp.Check#instantiatePoly` 方法，

```java
Type newpt = t.qtype.tag <= VOID ? t.qtype : syms.objectType;
```

其中 `t` 指的是 `get` 的返回类型 `T`，`syms.objectType` 指的就是 `Object` 方法。

遍历方法，选择 `String.valueOf` 的过程在 `com/sun/tools/javac/comp/Resolve.java` 的 `findMethod` 中。`String.valueOf` 的每个方法的参数，都会与 `get` 的返回类型，即 `Object` 进行匹配检查，方法是 `com/sun/tools/javac/code/Types.java:isSubtype`。例如，检查 `String.valueOf(double)` 方法时，会检查 `double` 与 `Object` 是否匹配。匹配的逻辑是：

  * `double` 是否与 `Object` 相等
  * `Object` 是否是 `double` 子类型

最终，会选择 `String.valueOf(Object)`。

而在 JDK8 中，问题变得比较复杂。

JDK8 入口为 `com.sun.tools.javac.Main#main`

JDK8 中不会推出 `get` 方法的返回类型为 `Object`，而是先设置一个 `DeferredAttr.DeferredType`。然后遍历 `String.valueOf` 的每个方法。假如当前检查的是 `String.valueOf(double)`。那么结合 `double` 和 `get` 的返回类型 `T`，推出 `T` 的类型为 `Double`。这个过程，最终会在 `com.sun.tools.javac.comp.Infer#generateReturnConstraintsPrimitive` 中体现，返回的是 `double` 的装箱类型 `Double`。

之后会检查：

* `double` 是否与 `Double` 相等。
* `Double` 是否是 `double` 的子类型。

这个过程会在 `com.sun.tools.javac.comp.Check#checkType(com.sun.tools.javac.util.JCDiagnostic.DiagnosticPosition, com.sun.tools.javac.code.Type, com.sun.tools.javac.code.Type, com.sun.tools.javac.comp.Check.CheckContext) ` 函数中的 `checkContext.compatible` 进行检查。

这时候，会发现，只有 `String.valueOf(char[])` 和 `String.valueOf(Object)` 满足了条件。并且 `char[]` 是 `Object` 的子类型，即是更具体的类型。所以，最终选择了 `String.valueOf(char[])`。检查更具体类型的代码在 `com.sun.tools.javac.comp.Resolve#mostSpecific`，最终调用的也是`com.sun.tools.javac.comp.Check#checkType(com.sun.tools.javac.util.JCDiagnostic.DiagnosticPosition, com.sun.tools.javac.code.Type, com.sun.tools.javac.code.Type, com.sun.tools.javac.comp.Check.CheckContext) ` 函数方法。检查 `Object` 是否是 `char[]` 的子类型，`char[]` 是否是 `Object` 的子类型。以此判断哪个类型更为具体。

不过在最开始，我们看字节码时，看出在 JDK8 中，`get()` 返回的类型应该与 JDK6 中一样，都是 `Object`, 不过在上述的语义分析过程中，看出 `get()` 返回的是 `char[]`。所以接下来，我们看在编译的其它阶段是不是发生了什么。

在此之前，我们先来看一下编译过程的入口, `com.sun.tools.javac.main.JavaCompiler#compile2`。

下面的代码是编译的入口:

```java
generate(desugar(flow(attribute(todo.remove()))));
```
我们刚才讲的都是在 `attribute` 函数中发生的过程。

接下来，进入 `flow` 函数，在由 attribute 函数生成的树上做数据流分析，主要分成四个功能：

 * AliveAnalyzer: 语句是否可达
 * AssignAnalyzer: 变量使用时已经赋值；final 变量没有被多次赋值
 * FlowAnalyzer: checked 异常是否被声明及抛出
 * CaptureAnalyzer: lambda 表达式或者内部类引用的局部变量要是 final

然后是 `desugar`，看起来是不是挺像解除语法糖的。这个过程会对类进行变换。把 `get()` 的返回类型由 `char[]` 变为 `Object` 的过程，就在这个函数中。最终是由 `com.sun.tools.javac.tree.TreeTranslator#translate(T)` 这个函数完成。

其中函数 `get` 会被变换：

```java
    public static <T> T get() {
        return (T) "x";
    }
```

会变成 

```java
    public static Object get() {
        return (Object) "x";
    }
```

同时，`String.valueOf(get())` 中的 `get()` 方法，其类型 `char[]` 也会变成 `Object`。这个过程在 `com.sun.tools.javac.comp.TransTypes#retype` 中完成。

```java
// tree: get(); erasedType: java.lang.Object; target: char[]
JCExpression retype(JCExpression tree, Type erasedType, Type target) {
..
}
```
由于 `get()` 返回类型为 `T`，所以这个函数会把 `T` 换成 `Object`，同时插入一条指令，将 `Object` 转换成 `char[]`。

`retype` 的本意如下面这个例子所示：

```
class Cell<A> { A value; }
Cell<Integer> cell;
Integer x = cell.value;
```

此时，会将 `cell.value` 返回值设置成 `Object`, 并且插入强制类型转换的指令。

但是在我们这次分析的情况中，`get()` 的返回类型 `T` 已经被推断出是 `char[]`，为什么又因为其定义是 `T`，然后被擦除，变为 `Object` 呢？这样一来，`String.valueOf` 选择了 `String.valueOf(char[])`，而 `get()` 的类型是 `Object`，又要强制转化成 `char[]`。既然如此，为什么不选择 `String.valueOf(Object)` 呢？感觉像个 bug 啊。我现在也不能理解为什么要这么做。

最后是 `generate`，生成字节码。生成 `get()` 字节码的源代码位于 `com.sun.tools.javac.jvm.Gen#genExpr`，此时参数 `tree` 为 `get()`，`pt` 为 `char[]`。生成 `get()` 字节码时，其返回类型已为 `Object`，而非 `char[]`。

最后总结一下文中最开始时提到的两个问题：

  * 对 `get()` 返回类型的推断
  * 对 `String.valueOf` 这个重载方法的选择

在 JDK6 中

  * `get()` 返回类型直接设置为 `Object`
  * `String.valueOf` 选择了 `String.valueOf(Object)`
  * `String.valueOf` 与 `get()` 匹配

在 JDK8 中

  * `get()` 返回类型先设置为 `DeferredAttr.DeferredType`
  * 遍历 `String.valueOf` 的方法，在满足条件的 `String.valueOf(Object)` 和 `String.valueOf(char[])` 中选择更为具体的 `String.valueOf(char[])`，`get()` 的返回类型也为 `char[]`
  * `get()` 的返回类型在 `desugar` 阶段被擦除，设置为 `Object`，同时强制转为 `char[]`
     
