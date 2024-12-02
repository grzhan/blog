title: 【Go 语言编程珠玑】分治法快速计算二进制数多少个 1
date: 2024-11-25 15:43:06
tags: 
  - Golang
  - Go 语言编程珠玑
  - 位运算
  - Hacker's Delight
  - 分治法
categories: Golang
thumbnailImage: gopher_group.png
---

{% asset_img "gopher_group.png" %}


在读 Golang GC 部分源码时候看到，对于 `mspan` 用于标记是否需要 GC 清除的 bitmap `mspan.gcmarkBits`，在 sweep 清扫过程中会使用 [countAlloc](https://github.com/golang/go/blob/733df2bc0af0f73d7bc9ee49a0d805b010293212/src/runtime/mgcsweep.go#L663)(`sys.OnesCount64`) 来快速计算 64 位二进制数中 1 的数量（即已分配的 object 数量）。

而 Golang 标准库的 `sys.OnesCount64` 是这么实现的：

```go
const m0 = 0x5555555555555555 // 01010101 ...
const m1 = 0x3333333333333333 // 00110011 ...
const m2 = 0x0f0f0f0f0f0f0f0f // 00001111 ...

func OnesCount64(x uint64) int {
    // Implementation: Parallel summing of adjacent bits.
	// See "Hacker's Delight", Chap. 5: Counting Bits.
	// The following pattern shows the general approach:
	//
	//   x = x>>1&(m0&m) + x&(m0&m)
	//   x = x>>2&(m1&m) + x&(m1&m)
	//   x = x>>4&(m2&m) + x&(m2&m)
	//   x = x>>8&(m3&m) + x&(m3&m)
	//   x = x>>16&(m4&m) + x&(m4&m)
	//   x = x>>32&(m5&m) + x&(m5&m)
	//   return int(x)
	//
	// Masking (& operations) can be left away when there's no
	// danger that a field's sum will carry over into the next
	// field: Since the result cannot be > 64, 8 bits is enough
	// and we can ignore the masks for the shifts by 8 and up.
	// Per "Hacker's Delight", the first line can be simplified
	// more, but it saves at best one instruction, so we leave
	// it alone for clarity.
	const m = 1<<64 - 1
	x = x>>1&(m0&m) + x&(m0&m)
	x = x>>2&(m1&m) + x&(m1&m)
	x = (x>>4 + x) & (m2 & m)
	x += x >> 8
	x += x >> 16
	x += x >> 32
	return int(x) & (1<<7 - 1)
}

```

和 [上一篇](https://grzhan.tech/2024/11/11/GoTrailingZero/) 提到的类似，也是通过一系列神秘的位运算实现了计数的效果。源码注释里提到，该方法来自 [Hacker's Delight](https://en.wikipedia.org/wiki/Hacker%27s_Delight) 第五章，为了整明白这个专门买了这本的实体书，这本书国内翻译叫[《算法心得-高效算法的奥秘》](https://item.jd.com/10103915205803.html)，很神奇不知道为啥要翻译成这个名字，其实这本书主要讲的不是计算机算法（computer algorithm），而是计算机算术（computer arithmetic）的“奇技淫巧”，在阅读诸多语言标准库的位运算相关函数时，能够作为重要的参考。

在《Hacker's Delight》的第五章 "位计数 (CountingBits)" 开头就讲到了这个二进制 1 个数的计算方法，就是运用“分治”（Divide and Conquer）策略，以 32 位二进制数`10111100011000110111111011111111`为例，我们采用如下图的分治策略来进行 1 的计数：

{% asset_img "ones_count_example.png" %}

第一步，我们将 32 位二进制数的位数每相邻两个组成一组，然后**我们将每组内的两个数相加，得到这个二进制数俩俩一组内每一租的 1 的 个数**，如上图 L2 所示，我们将`10111100011000110111111011111111`（L1）每相邻两位组成一组，然后将组内的数字相加：第一组是最高两位数 1 和 0，相加得到 01 ，第二组是第 3 位数 1 与第 4 位数 1，相加得到 10…… 就这样我们得到图中 L2 的 16 组值，**每组代表对应两位数中 1 的个数**。

第二步，我们将这 16 组数每相邻两组合并组成新的一组，每一组有 4 位数，其值同样是对应两组相加，如上图 L3 所示，01 + 10 得到 0011， 10 + 00 得到 0010 …… 这里我们可以发现，L3 的每一组四位二进制数，其实代表的就是**L1 每四位二进制数中 1 的个数**。L3 第一组 0011 对应 L1 第 1 位到第 4 位二进制 1 的个数，也就是 3 个；L3 第二组 0010 对应 L1 第 5 位到第 8 位 1 的个数，也就是 2 个…… 每次相加合并就是统计对应组 1 的个数的过程。

后续的第三、四、五、六步同理，就是合并相加统计对应组二进制 1 个数的过程，到达 L5 时，第一组对应就是 L1 第 1 位到第 16 位二进制 1 的个数，第二组对应就是 L1 第 17 位到第 32 位二进制 1 的个数。而这两组相加得到 L6，即整个 32 位二进制的 1 的个数。

整个分治的过程用代码来实现是这样的：

```go
x = (x & 0x55555555) + ((x >> 1) & 0x55555555) // 0x55555555 = 01010101....
x = (x & 0x33333333) + ((x >> 2) & 0x33333333) // 0x33333333 = 00110011....
x = (x & 0x0F0F0F0F) + ((x >> 4) & 0x0F0F0F0F) // 0x0F0F0F0F = 00001111....
x = (x & 0x00FF00FF) + ((x >> 8) & 0x00FF00FF) 
x = (x & 0x0000FFFF) + ((x >> 16) & 0x0000FFFF) 
```

核心思想就是刚才我们图示的分治算法，第一行就对应我们 L1 分组后合并得到 L2 的操作，这里的 `(x >> 1) & 0x55555555` 可能换成 `(x & 0xAAAAAAAA) >> 1` 会更容易理解，也就是把奇数位清 0 后再向右移 1 位与 `(x & 0x55555555)` 相加，得到对应 L2 的值，以此类推。

而之所以第一行用 `(x >> 1) & 0x55555555` 而非 `(x & 0xAAAAAAAA) >> 1` ，是因为这么做可以避免寄存器中生成两种大常数，如果对应计算机架构没有 "and not" 指令，那么就无法仅用 1 条指令基于 `0x55555555` 生成 `0xAAAAAAAA`。

上面的代码就是分治计算二进制 1 个数的基本原理了，而我们可以看到 `OnesCount64` 的源码还是有些不同，简化了后续的一些`与`操作，这里考虑到的优化有两点：

第一点是第三行运算 `x = (x>>4 + x) & (m2 & m)` 把 `& (m2&m)` 操作合并了，这是因为对应我们之前图例里 L3 到 L4 的合并操作，计算的是每组 8 位的二进制 1 个数，每组 8 位最大的可能值就是 8 ，所以 L3 的两组相加是不可能超过 4 位发生进位的，所以可以合并掉`与`操作。

第二点对应第四五六行的运算，类似地，不管如何相加，组内的有效值不会超过 7 位（1 的个数最多不会超过 64 个），因此我们只关心后续每组的末 7 位，这时候你就可以发现原先计算过程的高位清 0 已经没有必要，我们只要最后的时候统一把除了末 7 位以外的位数清 0 ，最终得到有效结果即可 —— 这也就是 `OnesCount64` 计算过程的全貌。

位运算的优化确实很有趣，也体现了过去这些天才工程师们的智慧。不过一般人确实很难理解这些优化算法是如何被这些人想出来的，无怪乎这本书叫做《Hacker's Delight》，只有真正聪明且狂热的人才会去追寻这种极致。


