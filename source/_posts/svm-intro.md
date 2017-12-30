---
title: Substrate VM 笔记
date: 2017-12-25 00:00:00
tags: SVM
---

# Introduction

  近期[开源][0]的Substrate VM（下称SVM）是一个构建在[Graal Compiler][1]上的，支持ahead-of-time (AOT) compilation的编译及运行框架。从执行时间上来看，SVM可划分为两部分：native image generator及SVM runtime。前者可看成一个普通的Java程序，用以生成可执行文件或动态链接库；后者是一个精简的runtime（与HotSpot相对应），包含异常处理，同步，线程管理，内存管理，Java Native Interface（JNI）等组件。SVM要求所运行的程序是封闭的，即不可动态加载其他类库等。这部分封闭的程序会被native image generator编译，并与SVM runtime相链接，继而形成一个自包含的二进制文件。

----

# Hello World

  我们在Oracle Technology Network（OTN）上迭代发布的[GraalVM][2]版本，其中便包含native image generator（``native-image``）。GraalVM附载的语言实现，如``js``，``ruby``，``python``，及``lli``（LLVM bitcode interpreter），乃至``native-image``本身，均是由该工具产生。下面我们将借助这一工具编译一个简单的Hello World程序：

```
$ echo "public class HelloWorld { public static void main(String[] args) { System.out.println(\"Hello World\"); } }" > HelloWorld.java
$ javac HelloWorld.java
$ /PATH/TO/GRAALVM_HOME/bin/native-image -H:Class=HelloWorld -H:Name=helloworld
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

  ``native-image``工具分为多个步骤，如[静态分析][3]，[编译][4]等。上述指令最终将在当前目录下生成指定名称（``-H:Name=``）的可执行文件，其入口为[``JavaMainWrapper.run``][5]。严格意义上讲入口是调用该方法的桩程序 —— SVM会为所有被[``@CEntryPoint``][6]注解的方法生成桩程序，完成由C空间调用至Java空间的准备工作。在``JavaMainWrapper.run``方法[调用][7]所指定主类（``-H:Class=``）的主方法之前并没有出现类似HotSpot中冗杂的初始化操作。实际上，``native-image``会维护一个[``NativeImageHeap``][8]（可看成在运行目标程序前Java堆的一个快照），并在生成二进制文件时直接烧录至其``__DATA``段中。因此，相较于HotSpot而言SVM运行Java程序的memory footprint更小。
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

  通过追加``-H:Debug=``参数，``native-image``生成的二进制文件会包含调试信息，有助于我们了解被链接的所有Java方法（下面罗列调用``JavaMainWrapper.run``的桩程序的多个别名）：

```
$ /PATH/TO/GRAALVM_HOME/bin/native-image -H:Class=HelloWorld -H:Name=helloworld -H:Debug=1
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

  ``native-image``的静态分析从一系列[root method][9]出发，保守地探索所有可被调用到的方法。如若探索到任一不被支持的功能，静态分析将抛出``UnsupportedFeatureException``，如``javac``的annotation processor所依赖的类加载机制。通过追加``-H:+ReportUnsupportedElementsAtRuntime``参数，``native-image``会将此类异常推延至运行时，使编译而成的``javac``支持不使用annotation processor的Java文件：

```
$ /PATH/TO/GRAALVM_HOME/bin/native-image -cp /PATH/TO/JAVA_HOME/lib/tools.jar -H:Class=com.sun.tools.javac.Main -H:Name=javac -H:+ReportUnsupportedElementsAtRuntime -H:IncludeResourceBundles=com.sun.tools.javac.resources.compiler,com.sun.tools.javac.resources.javac,com.sun.tools.javac.resources.version
   classlist:   2,395.38 ms
       (cap):   2,142.33 ms
       setup:   3,199.94 ms
  (typeflow):   7,480.60 ms
   (objects):   4,484.99 ms
  (features):      62.50 ms
    analysis:  12,374.56 ms
    universe:     874.61 ms
     (parse):   2,720.81 ms
    (inline):   2,563.05 ms
   (compile):  31,228.65 ms
     compile:  38,206.37 ms
       image:   6,352.41 ms
       write:   2,926.77 ms
     [total]:  66,438.46 ms
$ time ./javac -proc:none -bootclasspath /PATH/TO/JAVA_HOME/jre/lib/rt.jar HelloWorld.java
        0.05 real         0.02 user         0.01 sys
$ time javac HelloWorld.java
        0.59 real         1.09 user         0.11 sys
```

----

# HOWTO

  SVM源代码的编译需借助[``mx``][10]工具及支持``JVMCI``的JDK，如[labsjdk][11]（注：请下载页面最下方的labsjdk，直接使用GraalVM可能会导致共有类的编译问题）。下载完成后将``JAVA_HOME``指向labsjdk路径。

```
$ git clone https://github.com/graalvm/graal.git
$ cd graal/substratevm
$ mx build
$ mkdir svmbuild
$ echo "public class HelloWorld { public static void main(String[] args) { System.out.println(\"Hello World\"); } }" > svmbuild/HelloWorld.java
$ javac svmbuild/HelloWorld.java
$ mx image -cp $PWD/svmbuild -H:Class=HelloWorld -H:Name=helloworld
```

  指令``mx image``除自动将目标文件写入``svmbuild``目录外等同于``native-image``工具，例如同样需耗费较长的时间。SVM提供了另一种非缺省的解决方案，即让native image generator以server的形式运行（``mx image_server_start``），接受请求（通过向``mx image``追加``-server``参数）并返回编译结果。充分预热的native image generator，可将编译效率提升至三倍以上。
  与``javac``不同，对[Truffle][12]语言实现而言，由于运行需加载的Java类仅限于语言解释器及其依赖库，因此可以被SVM完美支持。下述指令将使用SVM编译ruby语言实现[TruffleRuby][13]：

```
$ cd /PATH/TO/graal/..
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

To be continued..

[0]: https://github.com/graalvm/graal/tree/master/substratevm
[1]: http://www.oracle.com/technetwork/oracle-labs/program-languages/overview/index.html
[2]: http://www.oracle.com/technetwork/oracle-labs/program-languages/downloads/index.html
[3]: https://github.com/graalvm/graal/blob/master/substratevm/src/com.oracle.svm.hosted/src/com/oracle/svm/hosted/NativeImageGenerator.java#L603
[4]: https://github.com/graalvm/graal/blob/master/substratevm/src/com.oracle.svm.hosted/src/com/oracle/svm/hosted/NativeImageGenerator.java#L780
[5]: https://github.com/graalvm/graal/blob/master/substratevm/src/com.oracle.svm.core/src/com/oracle/svm/core/JavaMainWrapper.java#L134
[6]: https://github.com/graalvm/graal/blob/master/sdk/src/org.graalvm.nativeimage/src/org/graalvm/nativeimage/c/function/CEntryPoint.java#L63
[7]: https://github.com/graalvm/graal/blob/master/substratevm/src/com.oracle.svm.core/src/com/oracle/svm/core/JavaMainWrapper.java#L155
[8]: https://github.com/graalvm/graal/blob/master/substratevm/src/com.oracle.svm.hosted/src/com/oracle/svm/hosted/image/NativeImageHeap.java
[9]: https://github.com/graalvm/graal/blob/master/substratevm/src/com.oracle.graal.pointsto/src/com/oracle/graal/pointsto/BigBang.java#L345
[10]: https://github.com/graalvm/mx
[11]: http://www.oracle.com/technetwork/oracle-labs/program-languages/downloads/index.html
[12]: https://github.com/graalvm/graal/tree/master/truffle
[13]: https://github.com/graalvm/truffleruby
