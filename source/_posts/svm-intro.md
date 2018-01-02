---
title: Substrate VM 笔记
date: 2017-12-25 00:00:00
tags: SVM
---

# Introduction

近期[开源][1]的Substrate VM（下称SVM）是一个构建在[Graal Compiler][2]上的，支持ahead-of-time (AOT) compilation的编译及运行框架。它的设计初衷是提供一个高startup performance，低memory print，以及能无缝衔接C代码（与JNI相较）的runtime，并能完美适配[Truffle][3]语言实现。

<!--more-->

这篇笔记将通过一些例子介绍SVM的实现及对应的源代码。如对SVM的架构，算法，性能等感兴趣，可参考如下链接：
* Overview: http://www.oracle.com/technetwork/java/jvmls2015-wimmer-2637907.pdf
* Video: https://www.youtube.com/watch?v=5BMHIeMXTqA
* Sources: https://github.com/graalvm/graal/tree/master/substratevm
* GraalVM: http://www.oracle.com/technetwork/oracle-labs/program-languages/
* Graal tutorial: http://lafo.ssw.uni-linz.ac.at/papers/2017_PLDI_GraalTutorial.pdf
* Publications: https://github.com/graalvm/graal/blob/master/docs/Publications.md

----

# Hello World

从执行时间上来看，SVM可划分为两部分：native image generator及SVM runtime。前者可看成一个普通的Java程序，用以生成可执行文件或动态链接库；后者是一个精简的runtime（与HotSpot相对应），包含异常处理，同步，线程管理，内存管理，Java Native Interface（JNI）等组件。SVM要求所运行的程序是封闭的，即不可动态加载其他类库等。这部分封闭的程序会被native image generator编译，并与SVM runtime相链接，继而形成一个自包含的二进制文件。

我们在Oracle Technology Network（OTN）上迭代发布的[GraalVM][5]版本，其中便包含native image generator（``native-image``）。GraalVM附载的语言实现，如``js``，``ruby``，``python``，及``lli`` （LLVM bitcode interpreter），乃至``native-image``本身，均是由该工具产生。下面我们将借助这一工具编译一个简单的Hello World程序：

```
$ export GRAALVM_HOME=/PATH/TO/GRAALVM/HOME
$ echo '// HelloWorld.java
public class HelloWorld {
    public static void main(String[] args) {
        System.out.println("Hello World");
    }
}' > HelloWorld.java
$ javac HelloWorld.java
$ $GRAALVM_HOME/bin/native-image -H:Class=HelloWorld -H:Name=helloworld
  classlist:   1,429.11 ms
      (cap):   2,010.48 ms
      setup:   3,088.00 ms
 (typeflow):   3,809.79 ms
  (objects):   2,962.38 ms
 (features):      32.76 ms
   analysis:   6,944.22 ms
   universe:     410.09 ms
    (parse):   1,072.11 ms
   (inline):     978.95 ms
  (compile):   9,421.44 ms
    compile:  12,207.86 ms
      image:   3,226.87 ms
      write:     797.77 ms
    [total]:  28,188.48 ms
```

``native-image``工具分为多个步骤，其中最重要的是[pointsto analysis][6]及[AOT compilation][7]。上述指令最终将在当前目录下生成指定名称（``-H:Name=``）的可执行文件，其入口为[``JavaMainWrapper.run``][8]。严格意义上讲入口是调用该方法的桩程序 —— SVM会为所有被[``@CEntryPoint``][9]注解的方法生成桩程序，完成由C空间调用至Java空间的准备工作。在``JavaMainWrapper.run``方法[调用][10]所指定主类（``-H:Class=``）的主方法之前并没有出现类似HotSpot中冗杂的初始化操作。实际上，``native-image``会维护一个[``NativeImageHeap``][11]（可看成在运行目标程序前Java堆的一个快照），并在生成二进制文件时直接烧录至其``__DATA``段中。因此，相较于HotSpot而言SVM运行Java程序的memory footprint更小。

``native-image``工具耗时较长，这是由于其实际工作是由一个运行在HotSpot上的子程序完成，因而受制于其启动性能较差的缺陷（解释执行）。之所以这部分代码没有被AOT编译，是因为native image generator依赖于HotSpot的类加载机制来载入需要编译的Java类。

编译而成的二进制文件的``__TEXT``段包含``HelloWorld.main``，JDK中所有可能被调用的方法，以及SVM runtime相应的代码。这部分runtime代码提供了简介中提及的各类runtime组件，但代价是生成的二进制文件大小远超于等价的helloworld.c编译而成的二进制文件：

```
// macOS-specific
$ size -m -x helloworld
Segment __PAGEZERO: 0x100000000
Segment __TEXT: 0x1f2000
	Section __stubs: 0xc
	Section __stub_helper: 0x24
	Section __text: 0x1f0014
	Section __const: 0x7e8
	Section __unwind_info: 0xa4
	total 0x1f08d0
Segment __DATA: 0x272000
	Section __nl_symbol_ptr: 0x10
	Section __got: 0x8
	Section __la_symbol_ptr: 0x10
	Section __const: 0x1f5aa0
	Section __data: 0x7a188
	total 0x26fc50
Segment __LINKEDIT: 0x3d7d4
total 0x1004a17d4
$ size -m -x helloworldc
Segment __PAGEZERO: 0x100000000
Segment __TEXT: 0x1000
	Section __text: 0x2a
	Section __stubs: 0x6
	Section __stub_helper: 0x1a
	Section __cstring: 0xd
	Section __unwind_info: 0x48
	total 0x9f
Segment __DATA: 0x1000
	Section __nl_symbol_ptr: 0x10
	Section __la_symbol_ptr: 0x8
	total 0x18
Segment __LINKEDIT: 0x1000
total 0x100003000
```

通过追加``-H:Debug=``参数，``native-image``生成的二进制文件将包含调试信息，有助于我们了解被链接的所有Java方法（下面罗列调用``JavaMainWrapper.run``的桩程序的多个别名）：

```
$ $GRAALVM_HOME/bin/native-image -H:Class=HelloWorld -H:Name=helloworld -H:Debug=1
$ objdump -t helloworld
...
0000000100006d90 g       __TEXT,__text  _com_002eoracle_002esvm_002ecore_002eJavaMainWrapper_002erun_0028ILorg_002egraalvm_002enativeimage_002ec_002etype_002eCCharPointerPointer_003b_0029
0000000100006d90 g       __TEXT,__text  _com_002eoracle_002esvm_002ecore_002ecode_002eCEntryPointCallStubs_002ecom_005f002eoracle_005f002esvm_005f002ecore_005f002eJavaMainWrapper_005f002erun_005f0028int_005f002corg_005f002egraalvm_005f002enativeimage_005f002ec_005f002etype_005f002eCCharPointerPointer_005f0029_0028int_002corg_002egraalvm_002enativeimage_002ec_002etype_002eCCharPointerPointer_0029
...
0000000100006d90 g       __TEXT,__text  _main
...
$ objdump -t helloworld | grep "_java." | wc -l
    1875
...
```

除了提供主函数生成可执行文件，我们还可以借助``-H:Kind=SHARED_LIBRARY``及``@CEntryPoint``来生成动态链接库，如下所示：

```
$ echo '// HelloWorld.java
import org.graalvm.nativeimage.c.function.CEntryPoint;
import org.graalvm.nativeimage.IsolateThread;

public class HelloWorld {
    @CEntryPoint
    public static void foo(IsolateThread thread) {
        System.out.println("Hello World");
    }
}' > HelloWorld.java
$ javac -cp $GRAALVM_HOME/jre/lib/boot/graal-sdk.jar HelloWorld.java
$ $GRAALVM_HOME/bin/native-image -H:Kind=SHARED_LIBRARY -H:Name=helloworld
```

上述指令将在当前目录下生成两个文件：动态链接库``helloworld.dylib``及对应的头文件``helloworld.h``。后者包含生成的入口函数及一系列用以操作SVM线程的API（暂无文档）。这两者的使用方式和常规动态链接库一致：

```
$ echo '// helloworld.c
#include "helloworld.h"

int main() {
    graal_isolate_t* isolate;
    graal_create_isolate(0, &isolate);
    HelloWorld_002efoo_0028Lorg_002egraalvm_002enativeimage_002eIsolateThread_003b_0029(graal_current_thread(isolate));
    return 0;
}' > helloworld.c
$ gcc -o helloworldc2svm helloworld.c helloworld.dylib
$ ./helloworldc2svm
Hello World
```

----

# Build from sources

SVM源代码的编译需借助[``mx``][13]工具及支持``JVMCI``的JDK，如[labsjdk][14]（注：请下载页面最下方的labsjdk，直接使用GraalVM可能会导致共有类的编译问题）。下载完成后将``JAVA_HOME``指向labsjdk路径。

```
$ git clone https://github.com/graalvm/graal.git
$ cd graal/substratevm
$ mx build
$ mkdir svmbuild
$ echo '// HelloWorld.java
public class HelloWorld {
    public static void main(String[] args) {
        System.out.println("Hello World");
    }
}' > svmbuild/HelloWorld.java
$ javac svmbuild/HelloWorld.java
$ mx image -cp $PWD/svmbuild -H:Class=HelloWorld -H:Name=helloworld
```

指令``mx image``除自动将目标文件写入``svmbuild``目录外等同于``native-image``工具，例如同样需耗费较长的时间。SVM提供了另一种非缺省的解决方案，即让native image generator以server的形式运行（``mx image_server_start``），接受请求（``mx image -server ...``）并返回编译结果。充分预热的native image generator，可将编译效率提升至三倍以上。

对Truffle语言实现而言，由于运行需加载的Java类仅限于语言解释器及其依赖库，因此可以被SVM完美支持。下述指令将使用SVM编译ruby语言实现[TruffleRuby][15]：

```
$ cd /PATH/TO/GRAAL/REPO/..
$ git clone https://github.com/graalvm/truffleruby.git
$ cd truffleruby
$ mx build
$ cd ../graal/substratevm
$ mx image -ruby
$ svmbuild/ruby -Xhome=../../truffleruby
```

编译TruffleRuby的指令较为复杂，因此我们提供了``mx image -ruby``的封装。它将在包含``graal``的git repo的路径下寻找``truffleruby``的repo，编译其通过``mx build``生成的jar包。

----

# SVM Internals

前段时间碰到一个很难复现的bug，极小概率在``graal-js``试图编译js代码时触发。经调试得知：1. 仅在native image中复现；2. 触发原因是``System.identityHashCode(Object)``对同一对象返回不同的值。在SVM中同一段代码可能被两个不同的runtime执行，即在生成二进制文件时由HotSpot运行（hosted mode），或在SVM runtime中运行。当时我推测``NativeImageHeap``中缓存了某些对象的identityHashCode（例如在``HashMap``中），而在SVM runtime中再对这些对象求identityHashCode则会得到不同的结果。但这一猜想被SVM组的开发者否定了，因为SVM已经有机制预防这种哈希值不一致的情况出现。

SVM无法直接使用HotSpot的runtime代码，因此所有的native方法皆需有对应的实现。SVM的解决方案是在Java层提供替代，并在编译过程中将编译内容替换掉。这些替代方法使用[``@TargetClass``][16]和[``@Substitute``][17]注解，如``System. identityHashCode(Object)``的替代：

```
// 修复前
@TargetClass(java.lang.System.class)
final class Target_java_lang_System {
  ...
  @Substitute
  private static int identityHashCode(Object obj) {
    if (obj == null) {
        return 0;
    }
    DynamicHub hub = KnownIntrinsics.readHub(obj);
    int hashCodeOffset = hub.getHashCodeOffset();
    if (hashCodeOffset == 0) {
        throw VMError.shouldNotReachHere("identityHashCode called on illegal object");
    }
    UnsignedWord hashCodeOffsetWord = WordFactory.unsigned(hashCodeOffset);
    int hashCode = ObjectAccess.readInt(obj, hashCodeOffsetWord);
    if (hashCode == 0) {
        // On the first invocation for an object create a new hash code.
        hashCode = ImageSingletons.lookup(IdentityHashCodeSupport.class).generateHashCode();
        ObjectAccess.writeInt(obj, hashCodeOffsetWord, hashCode);
    }
    return hashCode;
  }
  ...
}
```

该替代与[HotSpot的实现][18]非常类似，即从每个对象的特定偏移量读取其identityHashCode，如若不存在则随机生成一个。这段代码的问题在于没有对hashcode的写入操作加锁，因此使用CAS指令即可[修复][19]。

通过类似的机制native image generator可识别SVM所不支持的组件（类，字段，构造器，或方法）。这些组件需要由[``@Delete``][20]注解，或者通过``@Substitute``注解Java类并且不提供某些方法的替代来实现隐式标记。静态分析会从一系列[root method][12]出发，保守地探索所有可被调用到的方法。如若探索到任一不被支持的功能，它将抛出``UnsupportedFeatureException``。由于``javac``的annotation processor依赖于类加载机制，因此静态分析会探测到对``java.lang.ClassLoader.<init>``的调用，由这一[替代][21]隐式声明。

```
$ mx image -cp $JAVA_HOME/lib/tools.jar -H:Class=com.sun.tools.javac.Main -H:Name=javac -H:IncludeResourceBundles=com.sun.tools.javac.resources.compiler,com.sun.tools.javac.resources.javac,com.sun.tools.javac.resources.version
classlist:   3,167.32 ms
    (cap):   1,709.58 ms
    setup:   3,289.98 ms
 analysis:  12,421.88 ms
error: unsupported features in 3 methods
Detailed message:
Error: Bytecode parsing error: Unsupported constructor java.lang.ClassLoader.<init>(ClassLoader) is reachable: The declaring class of this element has been substituted, but this element is not present in the substitution class
To diagnose the issue, you can add the option -H:+ReportUnsupportedElementsAtRuntime. The unsupported element is then reported at run time when it is accessed the first time.
Trace:
        at parsing java.security.SecureClassLoader.<init>(SecureClassLoader.java:76)
Call path from entry point to java.security.SecureClassLoader.<init>(ClassLoader):
        at java.security.SecureClassLoader.<init>(SecureClassLoader.java:76)
        at java.net.URLClassLoader.<init>(URLClassLoader.java:100)
        ...
```

通过追加``-H:+ReportUnsupportedElementsAtRuntime``参数，native image generator会将此类异常推延至运行时，使编译而成的``javac``支持不使用annotation processor的Java文件：

```
$ mx image -H:+ReportUnsupportedElementsAtRuntime -cp $JAVA_HOME/lib/tools.jar -H:Class=com.sun.tools.javac.Main -H:Name=javac -H:IncludeResourceBundles=com.sun.tools.javac.resources.compiler,com.sun.tools.javac.resources.javac,com.sun.tools.javac.resources.version
   classlist:   3,442.31 ms
       (cap):   1,825.62 ms
       setup:   3,298.60 ms
  (typeflow):   7,639.63 ms
   (objects):   4,105.75 ms
  (features):     139.78 ms
    analysis:  12,633.02 ms
14 method(s) included for runtime compilation
    universe:     784.26 ms
     (parse):   2,098.70 ms
   (compile):  11,272.73 ms
     compile:  14,692.47 ms
       image:   3,576.05 ms
       write:   1,870.27 ms
     [total]:  40,407.83 ms
$ echo '// HelloWorld.java
public class HelloWorld {
    public static void main(String[] args) {
        System.out.println("Hello World");
    }
}' > HelloWorld.java
$ time svmbuild/javac -proc:none -bootclasspath $JAVA_HOME/jre/lib/rt.jar HelloWorld.java
        0.05 real         0.02 user         0.01 sys
$ time javac HelloWorld.java
        0.59 real         1.09 user         0.11 sys
```

To be continued..

[1]: https://github.com/graalvm/graal/tree/master/substratevm
[2]: http://www.oracle.com/technetwork/oracle-labs/program-languages/
[3]: https://github.com/graalvm/graal/tree/master/truffle
[4]: http://www.oracle.com/technetwork/database/multilingual-engine/
[5]: http://www.oracle.com/technetwork/oracle-labs/program-languages/downloads/index.html
[6]: https://github.com/graalvm/graal/blob/master/substratevm/src/com.oracle.svm.hosted/src/com/oracle/svm/hosted/NativeImageGenerator.java#L603
[7]: https://github.com/graalvm/graal/blob/master/substratevm/src/com.oracle.svm.hosted/src/com/oracle/svm/hosted/NativeImageGenerator.java#L780
[8]: https://github.com/graalvm/graal/blob/master/substratevm/src/com.oracle.svm.core/src/com/oracle/svm/core/JavaMainWrapper.java#L134
[9]: https://github.com/graalvm/graal/blob/master/sdk/src/org.graalvm.nativeimage/src/org/graalvm/nativeimage/c/function/CEntryPoint.java#L63
[10]: https://github.com/graalvm/graal/blob/master/substratevm/src/com.oracle.svm.core/src/com/oracle/svm/core/JavaMainWrapper.java#L155
[11]: https://github.com/graalvm/graal/blob/master/substratevm/src/com.oracle.svm.hosted/src/com/oracle/svm/hosted/image/NativeImageHeap.java#L77
[12]: https://github.com/graalvm/graal/blob/master/substratevm/src/com.oracle.graal.pointsto/src/com/oracle/graal/pointsto/BigBang.java#L345
[13]: https://github.com/graalvm/mx
[14]: http://www.oracle.com/technetwork/oracle-labs/program-languages/downloads/index.html
[15]: https://github.com/graalvm/truffleruby
[16]: https://github.com/graalvm/graal/blob/master/substratevm/src/com.oracle.svm.core/src/com/oracle/svm/core/annotate/TargetClass.java#L70
[17]: https://github.com/graalvm/graal/blob/master/substratevm/src/com.oracle.svm.core/src/com/oracle/svm/core/annotate/Substitute.java#L53
[18]: http://hg.openjdk.java.net/graal/graal-jvmci-8/file/8128b98d4736/src/share/vm/runtime/synchronizer.cpp#l634
[19]: https://github.com/graalvm/graal/blob/master/substratevm/src/com.oracle.svm.core/src/com/oracle/svm/core/jdk/JavaLangSubstitutions.java#L514
[20]: https://github.com/graalvm/graal/blob/master/substratevm/src/com.oracle.svm.core/src/com/oracle/svm/core/annotate/Delete.java#L38
[21]: https://github.com/graalvm/graal/blob/master/substratevm/src/com.oracle.svm.core/src/com/oracle/svm/core/jdk/JavaLangSubstitutions.java#L244
