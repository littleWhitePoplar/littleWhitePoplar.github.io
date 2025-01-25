---
layout: post
title: "Why does unordered_map perform so slow"
date: 2025-01-25 00:00:00 +0000
categories: jekyll update
---

## Analysis of slow performance of unordered_map

unordered_map性能低的原因主要在于三点：
* 更多的指令数
* 更高的dcache miss
* 更高的dtlb miss

<p align = "center">    
<img  alt="second-and-inst-and-ipc" src="{{site.baseurl}}/assets/images/2025-01-25_second-and-inst-and-ipc.png" width="400" />
</p>

<p align = "center">    
<img  alt="dcache-miss-and-tlb-miss" src="{{site.baseurl}}/assets/images/2025-01-15_dcache-miss-and-tlb-miss.png" width="400" />
</p>

为了得到上述的数据，以下是我准备的一些内容：

CPU: Xeon Gold 6161  
gcc: 9.4.0  
benchmark:   udb2(https://github.com/attractivechaos/udb2.git)

### 1. 源码分析

首先，unordered_map是一种链式的散列表，它首先会分配n个桶，当冲突时，新的元素也会被加入到桶所在的链表中。

默认初始大小: 11  
默认扩容策略：当前桶大小*2的后一个质数，例如当前为11，扩容时桶大小会变为23  
默认最大加载因子：1.0，即当每个桶的平均元素数超过该阈值，会进行扩容  

下面的几张图，分别是unordered_map的初始化，到分别添加14、25、12、23这几个键后的状态。

> 这里没有贴源码，是为了让大家简要了解它内部的大致实现，而不是分析源码中的具体优化和实现细节。

![ini]({{site.baseurl}}/assets/images/2025-01-25_ini.svg)

![emplace_14]({{site.baseurl}}/assets/images/2025-01-25_emplace_14.svg)

![emplace_25]({{site.baseurl}}/assets/images/2025-01-25_emplace_25.svg)

![emplace_12]({{site.baseurl}}/assets/images/2025-01-25_emplace_12.svg)

![emplace_23]({{site.baseurl}}/assets/images/2025-01-25_emplace_23.svg)

从上面的图可以看到，每个桶实际指向了它的第一个元素的前一个节点，这是因为为了方便去进行insert和erase操作，因为这两个操作是需要知道前一个节点的。  

举个例子，当查询键12时：
1. 12 % 11 == 1，这时我们找到了桶1的指针。
2. 比较桶1的next和12是否一致，如果一致，我们返回它；否则继续比较键23的next和12是否一致...一直到找到这个键，或者找到桶id != 1。

### 2. perf分析

为了清楚的得知unordered_map的瓶颈，我们使用perf记录程序的热点信息。  

这里我们借助udb2这个benchmark: 

```bash
git clone https://github.com/attractivechaos/udb2.git
cd udb2/c++11
# 为了方便分析，需要在Makefile中添加-g参数
make -j
perf record --call-graph lbr ./run-test
perf report
```

这里可以看到，test_int这个函数占了接近90%的运行时间。它占比最高的指令是下面几条:

1. mov (%rdi),%rbp

该汇编占据了test_int的20.49%时间，实际对应的是下面的c++语句，它实际是把前一个节点的next赋值给__p。它为什么会如此的消耗性能呢？原因很简单，从源码分析的那几个图，我们能看到内部是以链表的形式来实现的，这会导致分配的每个节点的缓存性不友好，也就是dcache和dtlb容易miss。
```
__node_ptr __p = static_cast<__node_ptr>(__prev_p->_M_nxt);
```

2. mov 0x10(%rbp),%rsi

该汇编占据了testint的39%的时间，实际对应的是下面的两句c++语句:
```cpp
this->_M_equals(__k,__code,__p);
_M_bucket_index(__p->M_next());
```
，它实际是取出__p的key，用来和查询的key进行对比。它消耗性能的原因有两点:
* 缓存不友好，不重复进行解释。

* 重复的取出同一个节点的key。循环中的_M_bucket_index()先去查询当前节点的下一个节点的key，然后使用key % bucket_count，来得到bucket_index，如果这个节点还是当前桶的，下一次循环进来，执行this->_M_equals时，又会去查询__p的key。这里就造成了重复的解指针的行为。(如果这里没看懂，查看源码分析中的最后一幅图，如果查询的是  34这个键，当你查到键12的时候，会发现25实际已经是属于桶3了)。  
但同时，如果之前已经查询过了，现在再去查询已经缓存到了，也仅仅是一条mov指令，应该不会有缓存相关的问题，所以这里是不会有太大的性能问题。

```cpp
for (__node_ptr __p = static_cast<__node_ptr>(__prev_p->_M_nxt);;
	 __p = __p->_M_next())
{
	if (this->_M_equals(__k, __code, *__p))
		return __prev_p;

	if (!__p->_M_nxt || _M_bucket_index(*__p->_M_next()) != __bkt)
		break;
	__prev_p = __p;
}
```
