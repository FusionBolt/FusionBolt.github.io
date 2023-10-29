---
title: mold源码阅读十六 回顾整个流程
typora-root-url: ../../source
date: 2023-08-05 18:17:42
category: Linker
tags: mold
---

![Untitled](/images/mold-16-summary/Untitled.png)

<center style="font-size:14px;color:#C0C0C0;text-decoration:underline">pixiv:80173499</center> 

# 内容回顾

在以往十六期的博客中，我们沿着mold中的main函数一路追寻了下去，直到结束。

首先我们熟悉了[文件结构以及项目目录](https://homura.live/2023/02/12/mold/mold-0/)等，[查看了如何读取不同类型的文件](https://homura.live/2023/02/26/mold/mold-1-read-input-files/#read-file)，其中最关键的是[obj](https://homura.live/2023/02/26/mold/mold-1-read-input-files/#ObjectFile)，[dso](https://homura.live/2023/04/05/mold/mold-2-read-shared-files/)，lto三种，分析了不同类型的差异及其特有的处理方式。同时在查看如何解析elf的过程中了解elf头，了解了[mold对象的构造](https://homura.live/2023/02/26/mold/mold-1-read-input-files/#InputFile)，以及elf中信息查找段、符号等的方式。

当我们收集齐输入的文件信息后就要开始对这些文件进行处理。首先做的是[dso去重，避免多个同名dso导致的错误](https://homura.live/2023/04/09/mold/mold-3-symbol-resolve/#dso-uniquely)。接下来是最重要部分：符号信息解析。符号相关的过程有许多包括[符号决议](https://homura.live/2023/04/09/mold/mold-3-symbol-resolve/#符号决议)，[符号的导入导出](https://homura.live/2023/04/29/mold/mold-5-symbol/#compute-import-export)，[动态链接的符号的版本确定](https://homura.live/2023/04/29/mold/mold-5-symbol/)，[处理未解析的符号](https://homura.live/2023/06/19/mold/mold-9-unresolve-symbol/#claim-unresolved-symbols)等等。之后还对[mergeable的段进行合并](https://homura.live/2023/04/16/mold/mold-4-mergeable-section/)。

处理完符号相关的信息后开始[创建输出文件](https://homura.live/2023/06/10/mold/mold-8-create-output-section/#create-output-sections)，准备将许多input section合成一个output section。此时需要对于常见synthetic[符号](https://homura.live/2023/06/10/mold/mold-8-create-output-section/#add-synthetic-symbols)与段的构造，并且将它们放到输出的文件中。在实际输出文件之前还需要确定文件内部布局，主要对[段排序](https://homura.live/2023/06/24/mold/mold-10-sort-section/)，其中包括chunks之间排序，以及output section内部保存的input section的顺序。

当文件布局确定后我们就可以[创建rel相关的段，将符号写入对应的符号表](https://homura.live/2023/07/02/mold/mold-11-rel-and-dynsym/)，计算段内的一些信息，对部分[synthetic段构造](https://homura.live/2023/05/20/mold/mold-7-before-create-output-section/#Create-Synthetic-Sections)等等。之后[更新section对应的shdr](https://homura.live/2023/07/15/mold/mold-13-compute-shdr-and-set-osec-offsets/#compute-section-headers)，以及[更新段的虚拟地址](https://homura.live/2023/07/15/mold/mold-13-compute-shdr-and-set-osec-offsets/)，对[synthetic符号的值进行修正](https://homura.live/2023/07/26/mold/mold-14-fix-file-layout-and-create-output/#fix-synthetic-symbols)等。通过这些操作来确定下文件载入内存中的布局。

最后再将这些[拷贝到输出文件](https://homura.live/2023/07/26/mold/mold-14-fix-file-layout-and-create-output/#copy-chunks)中。在拷贝的同时还做了许多操作，比如说段的重定位，填写ehdr以及其他synthetic段中的信息。

在整个过程中也有许多检查，比如符号重复定义或者未定义。还有许多优化，例如对[section进行标记来回收无用段](https://homura.live/2023/05/07/mold/mold-6-section-size-reduce/#gc-sections)，[安排输出段的位置](https://homura.live/2023/07/15/mold/mold-13-compute-shdr-and-set-osec-offsets/#set-virtual-addresses-by-order)使得相同读写权限的段尽可能在一个页内，[消除重复的ehframe项](https://homura.live/2023/05/07/mold/mold-6-section-size-reduce/#icf-sections)，段压缩等等。

以上就是链接器mold的概况。

# 想法

最初会开始看链接器的实现是因为感到好奇，加上之前每次遇到链接相关的问题第一反应是头大，觉得解决不了。后来看到mold这个链接器，其中的代码量还在我能串一遍理解的范畴，因此开始了读代码的过程。读的时候做着记录，后来想着干脆开一个系列博客，我在读的时候经常容易跳过某些细节，写博客的过程中会强制自己对这些细节进行强制思考。

经过了十六期的文章后，整个mold的链接过程基本上就全部过了一遍，而我对于链接器工作的整个流程有了更详细的认知。以前对于链接的模糊印象就是简单的相似段合并，符号解析（但是不知道符号解析具体是在做什么），生成可执行文件或者library，但现在我对于这些部分有了更多的了解，并且还知道了链接过程不止有这些，还有包括synthetic的符号和段的处理，虚拟地址计算，重定位操作等等。

除此之外还看到了许多未曾想到的东西，在看到一些处理过程后，对动态链接以及加载的过程也有了更多的了解，还有一些之前从未想过能如何联系到一起的想法，比如说相同attribute的段放在一起，避免单独成页，减少运行时的内存等。

虽然学到了很多东西但是还是有很多地方其实是一知半解，阅读源码远不如实际写来困难，虽然能够大致讲出整个链接器的结构是怎样的，但是对链接器来说最重要的还是各种边边角角的细节，或者意想不到的东西都会在写的过程中出现。我现在在造各种轮子玩，想自己做出各种东西并且串联起来，或许会有一天也会需要造自己的linker吧。

在源码通读的过程可能花了过久的时间，有些低效。但很多东西我一开始确实没意识到，很多问题都没有提出，不过查看了前面的这些过程后，现在开始阅读不仅是了解了有什么，还让我能够提出一些问题。

在博客内容写作的过程也不太熟练，最近也是为自己博客写作感到焦虑。不论是内容详细程度，以及排版，内容划分做的都不太好。在学习的时候看到maskray聚聚的文章，多少受到了一些启发，意识到自己过于注重于原来的代码怎么写，对于代码背后的原理关注的相对较少，这其实才是要学习的本质内容，又不是学习代码技巧。在后面几期也在有意识的进行改正，之后写其他阅读代码的博客时也会继续试着这样来做。

排版上试了不同的方案，比如说最早是放一段读一段，后来又尝试一次讲完一个流程然后贴整段代码，之后再对其中需要更加深入的细节加入小标题来做。不过感觉怎么都很别扭，markdown似乎没什么办法分成代码和文字两列，最后还是觉得先讲清楚流程再放代码了。

# 后续

如果后续勤快的话还会继续更新一些东西，除了这样通读外，还想针对特定主题进行贯穿一遍，而且还有一些没有详细看细节但是比较重要的东西。（总觉得说出这样的话就会懒得更了…）

比如说各种synthetic的符号更详细的介绍， 梳理做的各种优化，header的生成，为动态链接做的准备（got，plt等），数据压缩与解压，为重定位所做的各种操作，最终产物的地址计算与关联等等，这些其实都还比较模糊，没有一个确切的印象，需要单独串联起来理解整个过程。
