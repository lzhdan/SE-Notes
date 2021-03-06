# JVM相关知识总览
***
## 简介
&ensp;&ensp;&ensp;&ensp;对所学的JVM虚拟机知识做一个总结，尽量把他们串起来

## 概览
&ensp;&ensp;&ensp;&ensp;知识大致可以分为下面的几个部分：

- Java语言的相关特性：如自动GC、跨平台等等，这部分有个大致了解即可
- 字节码：有点汇编的感觉，但如果不是搞虚拟机的，也只需要有个大致的了解即可
- 类加载相关：JVM的类加载过程；类加载时机；这部分需要掌握，可以说是基础，后面的反射、AOP相关概念和这块有些关联，掌握它利于后面的运用
- JVM运行参数：需要有个大致的了解
- 内存模型：需要熟练掌握JVM的内存分配模型，因为涉及到后面的GC、调优等
  - GC算法及原理：需要进行掌握，用于调优和排查问题心中有数
  - GC工具：需要有个大致了解和掌握（还是需要平时多练）
  - 调优思路：这块感触不是太深，工作中这方面玩的太少了

&ensp;&ensp;&ensp;&ensp;Java语言有两大特色：跨平台和自动垃圾回收

&ensp;&ensp;&ensp;&ensp;Java如何做到跨平台：通过加了一个中间层--JVM虚拟机进行实现的。所以我们需要对JVM的加载运行机制有所了解，其中就涉及到了字节码和类加载过程

&ensp;&ensp;&ensp;&ensp;字节码部分不是重点就跳过

&ensp;&ensp;&ensp;&ensp;类加载过程需要有详细的了解，其中也涉及到了一些比较重要的知识点：JVM加载过程、类触发加载时机等待。其中还涉及到了代码层次的运用：类加载器、类的加解密加载等等，相关的细节文章如下：

- [JVM类加载:Java类的生命周期](https://juejin.cn/post/6927196670044160013/)
- [JVM类加载器](https://juejin.cn/post/6927246702101397512/)

&ensp;&ensp;&ensp;&ensp;虽然写代码不用自己进行内存的分配和管理，但需要对其原理有了解，以便于在出现问题的时候进行排查和性能调优

&ensp;&ensp;&ensp;&ensp;首先得对JVM的内存模型有个清晰的认识，对其进行一个总结：

- [JVM内存模型](https://juejin.cn/post/6927414800376922126/)

&ensp;&ensp;&ensp;&ensp;GC是对堆内存进行回收处理，我们需要了解大致的GC算法原理：

- [Java内存GC算法](https://juejin.cn/post/6927415686654328839/)

- CMS最大yong区堆内存：64（操作系统位数） * 4（并行GC线程数） * 13 / 10

&ensp;&ensp;&ensp;&ensp;JVM启动参数，做个记录，需要用时可以回来查阅

- [Java JVM 启动参数](https://juejin.cn/post/6927426873051840520/)

&ensp;&ensp;&ensp;&ensp;有相关的工具，能帮助我们更加快速和便捷的排除分析问题(搬运工，简单记录）：

- [Java相关工具](https://juejin.cn/post/6927427980603949063/)

&ensp;&ensp;&ensp;&ensp;最后，JVM调优经验（真搬运工，希望遇到问题的时候能想起）：

- [Java调优相关](https://juejin.cn/post/6927436352548143111/)

## 实践作业相关
PS:没有链接的就是重要的，应该做的，但没做的......，后面补补

- [自定义类加载器](https://github.com/lw1243925457/JAVA-000/tree/main/Week_01)
- [GC性能简单测试](https://github.com/lw1243925457/JAVA-000/blob/main/Week_02/%E4%BD%9C%E4%B8%9A.md)
- 1.20-实现xlass打包的xar（类似class文件打包的jar）的加载：xar里是xlass
- 2.30-基于自定义Classloader实现类的动态加载和卸载：需要设计加载和卸载
- 3.30-基于自定义Classloader实现模块化机制：需要设计模块化机制
- 4.30-使用xar作为模块，实现xar动态加载和卸载：综合应用前面的内容

## 参考资料
- Java进阶训练营
- 《深入理解Java虚拟机》 周志明 第三版
- 《深入拆解Java虚拟机》 郑雨迪 极客时间