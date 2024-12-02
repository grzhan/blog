title: 【Go 语言设计与实现】 笔记 — Select 源码分析
date: 2024-06-25 21:08:46
tags:
	- Golang
	- Golang Internal
	- Select
categories: Golang
thumbnailImage: gopher_select.png
---

{% asset_img "gopher_select.png" %}


左书祺老师的[《Go语言设计与实现》](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-select/)写得很好，不过在阅读过程中发现不少部分还是要结合阅读源码才能够理解其中细节。

与之前的笔记类似，本篇将围绕 [5.2 select](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-select/) 一节通过查看资料和阅读源码进行整理补充，以免自己回头忘记。

也欢迎熟悉这部分源码或是感兴趣的老板参与讨论。

基于阅读的 Golang 源码版本是 1.16

原书中对于 select 源码的分析事实上是从两个层面来分析的，第一部分是 Go 语言编译器层面（`src/cmd/compile/internal/gc/*`），介绍 Go 编译器（也叫 gc）如何将 go 代码文件中的 `select ... case ...` 语句进行编译的，对应分析的函数是 `walkselectcases` （`src/cmd/compile/internal/gc/select.go`） ；第二部分是 Go 运行时层面，即 `select` 语句如何处理多个 `case` 的主要核心逻辑，这部分涉及到重要 runtime 函数调用 `selectgo`(`src/runtime/select.go`) 。本篇笔记也将主要介绍这两部分内容，其中编译器的部分主要聚焦大体上逻辑的理解，很多内容原书中已经提到过了这里可能是晒出源码呼应一下，而 `selectgo` 本文会重点过一下。

# 1. walkselectcases

在原书 [5.2.3 实现原理](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-select/)一节中， 介绍了编译器对于 select 语句在编译期间进行的处理：将 select 语句转换成 `OSELECT` 节点。每个 `OSELECT` 节点都会持有一组 `OCASE` 节点。而根据 select 中 case 的不同对控制语句进行优化，这一过程都在 `walkselectcases` 函数中。与原书中写的一样，`walkselectcases` 按四类情况处理 case：

1. select 不存在任何 case
2. select 只存在一个 case
3. select 存在两个 case， 其中一个是 default
4. select 存在多个 case

事实上这部分原书讲得挺清楚的，还给出了编译器转换后的等价代码（这是源码里没有的），前三部分这里主要贴出源码，和原书的介绍对照一下。

第四部分核心逻辑在 `runtime.selectgo` , 这部分在分析 `selectgo` 的时候再讲。

 `walkselectcases` 函数本身代码比较多（259行），这里还是按照场景分章节贴出相关代码

## 1.1 select 不存在任何 case

```go
func walkselectcases(cases *Nodes) []*Node {
	ncas := cases.Len()
	sellineno := lineno

	// optimization: zero-case select
	if ncas == 0 {
		return []*Node{mkcall("block", nil, nil)}
	}
	// ...
}

func block() {
	gopark(nil, nil, waitReasonSelectNoCases, traceEvGoStop, 1) // forever
}
```
（`src/cmd/compile/internal/gc/select.go`, 108-115 行）

这部分就是 select 语句里不存在任何 case 的情况， `ncas` 就是 case 数量，如果为 0 ，就会调用 `gopark`，在前面的 channel 的源码分析讲过 gopark，这个函数会将 goroutine 与处理器 M 解绑。像 channel 在 gopark 之前会把 goroutine 放在自己的等待队列里，以及信号量实现在 gopark 之前也会放到自己 treap 树堆的等待队列里，而这里没做类似的操作，也就是原书说的会让当前 goroutine 进入无法被唤醒的休眠状态（*其实我有点不理解为什么这么处理，为什么不是抛出错误，比如 panic*）

这里值得提一下的是 `mkcall` ， 这个函数会生成一个调用函数的 Node ，即编译阶段的 AST 语法树节点，而这里就是一个调用 `runtime.block` 函数的节点。

## 1.2 select 存在一个 case

```go
	// optimization: one-case select: single op.
	if ncas == 1 {
		cas := cases.First()
		setlineno(cas)
		l := cas.Ninit.Slice()
		if cas.Left != nil { // not default:
			n := cas.Left
			l = append(l, n.Ninit.Slice()...)
			n.Ninit.Set(nil)
			switch n.Op {
			default:
				Fatalf("select %v", n.Op)

			case OSEND:
				// already ok

			case OSELRECV, OSELRECV2:
				if n.Op == OSELRECV || n.List.Len() == 0 {
					if n.Left == nil {
						n = n.Right
					} else {
						n.Op = OAS
					}
					break
				}

				if n.Left == nil {
					nblank = typecheck(nblank, ctxExpr|ctxAssign)
					n.Left = nblank
				}

				n.Op = OAS2
				n.List.Prepend(n.Left)
				n.Rlist.Set1(n.Right)
				n.Right = nil
				n.Left = nil
				n.SetTypecheck(0)
				n = typecheck(n, ctxStmt)
			}

			l = append(l, n)
		}

		l = append(l, cas.Nbody.Slice()...)
		l = append(l, nod(OBREAK, nil, nil))
		return l
	}
```
（`src/cmd/compile/internal/gc/select.go`, 117-163 行）

这一段就是当 `ncase == 1` 即整个 select 语句只有一个 case 时，我们直接把 case 里的发送/接收语句解析出来。

要更好地理解这段代码，首先要看一下 select 语句和 case 语句的文法定义（https://go.dev/ref/spec#Select_statements）：

```bnf
SelectStmt = "select" "{" { CommClause } "}" .
CommClause = CommCase ":" StatementList .
CommCase   = "case" ( SendStmt | RecvStmt ) | "default" .
RecvStmt   = [ ExpressionList "=" | IdentifierList ":=" ] RecvExpr .
RecvExpr   = Expression .
```

这里我们遍历查看的 `cas` 对应文法里的 `CommCase` ， 而 `cas` 是 `gc.Node` 类型，要理解代码需要知道 `gc.Node` 对应 `CommCase` 其对应字段存了啥， 我们可以找到 `OCASE` 对应的说明（`src/cmd/compile/internal/syntax/token_string.go`）：

```go
	// OCASE:  case List: Nbody (List==nil means default)
	//   For OTYPESW, List is a OTYPE node for the specified type (or OLITERAL
	//   for nil), and, if a type-switch variable is specified, Rlist is an
	//   ONAME for the version of the type-switch variable with the specified
	//   type.
	OCASE
```

这里可以看出来，左操作数为 nil 时代表这个 `OCASE` 对应的是 `default` 语句，对于 `default` 代码就不需要做啥特殊处理，把对应 `cas.NBody` 里的语句加到 l 里返回出去就可以。所以整段代码主要还是看 `cas.Left != nil` 的情况，也就是看 `n`

如果 `n.Op` 是 `OSEND` 代表这是一个 channel 发送语句，没有什么特别处理的，直接把 n 给到 l 就可以 （`l = append(l, n)`），也就是整个 `select ... case ...` 语句变成了直接 send （如 `ch <- n` ）后面接着 case 代码块里面的语句就可以。

而 `OSELRECV`, `OSELRECV2` 代表 channel 接收的语句，情况会更复杂一些，我们先看下关于 `OSELRECV` 与 `OSELRECV2` 的说明：

```go
	OSELRECV     // Left = <-Right.Left: (appears as .Left of OCASE; Right.Op == ORECV)
	OSELRECV2    // List = <-Right.Left: (appears as .Left of OCASE; count(List) == 2, Right.Op == ORECV)
```

整体的代码就是为了将 case 里的接收语句直接抽出来和 case 代码块的相关语句放在一起，起到简化优化的作用。

原书中提供的该写代码提到了 `ch == nil` 然后 `block` 的条件判断，我在这部分源码倒是没看到，而且根据之前 channel 文章的分析，这个条件判断是在 runtime 函数 `chanrecv` 开始就有的。

> 这段代码我其实没看懂为什么要考虑 OCASE Node 对应 Ninit 的情况（`l = append(l, n.Ninit.Slice()...)`），因为根据文法，case 语句应该是没有初始化语句才对，当然编译器这块代码我没有通盘看下来理解还很浅，有知道的老板希望可以解惑一下，多谢！

## 1.3 select 存在两个 case， 其中一个是 default 

在处理这个场景之前，`walkselectcases` 还会做个操作，就是把 case 语句中的 Left 与 Right 对应的变量加上从值转换为地址的操作，同时把 default 语句从几个 case 中拎出来：

```go
	// convert case value arguments to addresses.
	// this rewrite is used by both the general code and the next optimization.
	var dflt *Node
	for _, cas := range cases.Slice() {
		setlineno(cas)
		n := cas.Left
		if n == nil {
			dflt = cas
			continue
		}
		switch n.Op {
		case OSEND:
			n.Right = nod(OADDR, n.Right, nil)
			n.Right = typecheck(n.Right, ctxExpr)

		case OSELRECV, OSELRECV2:
			if n.Op == OSELRECV2 && n.List.Len() == 0 {
				n.Op = OSELRECV
			}

			if n.Left != nil {
				n.Left = nod(OADDR, n.Left, nil)
				n.Left = typecheck(n.Left, ctxExpr)
			}
		}
	}
```
（`src/cmd/compile/internal/gc/select.go`, 165-190 行）


`dflt` 就是 default 语句，这段代码相当于给变量加了 '&', 也就是类似于 `ch <- &v` 、 `&v = <- ch` ，我想其实本来不加 '&' 而是直接使用变量在代码里本身也算是一种语法糖了，因为其实 `chansend` 与 `chanrecv` 通道的发送与接收本身就是把数据发给变量地址指向的内存，或者把变量地址指向的内存数据拿出来给通道。

后面就是处理两个 case， 其中一个是 default 的情况：

```go
	// optimization: two-case select but one is default: single non-blocking op.
	if ncas == 2 && dflt != nil {
		cas := cases.First()
		if cas == dflt {
			cas = cases.Second()
		}

		n := cas.Left
		setlineno(n)
		r := nod(OIF, nil, nil)
		r.Ninit.Set(cas.Ninit.Slice())
		switch n.Op {
		default:
			Fatalf("select %v", n.Op)

		case OSEND:
			// if selectnbsend(c, v) { body } else { default body }
			ch := n.Left
			r.Left = mkcall1(chanfn("selectnbsend", 2, ch.Type), types.Types[TBOOL], &r.Ninit, ch, n.Right)

		case OSELRECV:
			// if selectnbrecv(&v, c) { body } else { default body }
			ch := n.Right.Left
			elem := n.Left
			if elem == nil {
				elem = nodnil()
			}
			r.Left = mkcall1(chanfn("selectnbrecv", 2, ch.Type), types.Types[TBOOL], &r.Ninit, elem, ch)

		case OSELRECV2:
			// if selectnbrecv2(&v, &received, c) { body } else { default body }
			ch := n.Right.Left
			elem := n.Left
			if elem == nil {
				elem = nodnil()
			}
			receivedp := nod(OADDR, n.List.First(), nil)
			receivedp = typecheck(receivedp, ctxExpr)
			r.Left = mkcall1(chanfn("selectnbrecv2", 2, ch.Type), types.Types[TBOOL], &r.Ninit, elem, receivedp, ch)
		}

		r.Left = typecheck(r.Left, ctxExpr)
		r.Nbody.Set(cas.Nbody.Slice())
		r.Rlist.Set(append(dflt.Ninit.Slice(), dflt.Nbody.Slice()...))
		return []*Node{r, nod(OBREAK, nil, nil)}
	}
```
（`src/cmd/compile/internal/gc/select.go`, 192-237 行）

可以看到，这个优化最主要的操作就是创建了一个 `OIF` 节点 `r` ，这个 IF 语句会把通道操作语句作为条件语句，对应 case 的代码块作为条件成功的代码块，而 default 作为条件失败的代码块。在源码注释有展示这种转换，原书中也揭示了优化改变后的代码。

这里的 `selectnbsend` 、`selectnbrecv` 、`selectnbrecv2` 都是 `chanrecv` 与 `chansend` 的非阻塞调用，原书也有提及。

处理完上述几个 case 场景后，就是通用的场景了，这个通用场景也是整个 select 语句最核心的部分。


## 1.4 常见流程

### 1.4.1 初始化与预处理

在正式开始 `select ... case ...` 的正式流程之前，函数 `walkselectcases` 做了一系列初始化准备和预处理：

```go
	if dflt != nil {
		ncas--
	}
	casorder := make([]*Node, ncas)
	nsends, nrecvs := 0, 0

	var init []*Node

	// generate sel-struct
	lineno = sellineno
	selv := temp(types.NewArray(scasetype(), int64(ncas)))
	r := nod(OAS, selv, nil)
	r = typecheck(r, ctxStmt)
	init = append(init, r)

	// No initialization for order; runtime.selectgo is responsible for that.
	order := temp(types.NewArray(types.Types[TUINT16], 2*int64(ncas)))

```
（`src/cmd/compile/internal/gc/select.go`, 239-252 行）

首先就是把 `default` 的情况在后续踢出去， `ncas--`。然后就是初始化一系列要用的变量以及要新加的 AST 节点。

这里新建的节点里最重要的就是对应 `[ncas]scase` 类型的 `selv`，而对应 `scase` 的定义我们可以在 `src/runtime/select.go` 找到：

```go
// Select case descriptor.
// Known to compiler.
// Changes here must also be made in src/cmd/internal/gc/select.go's scasetype.
type scase struct {
	c    *hchan         // chan
	elem unsafe.Pointer // data element
}
```
这个结构体在 `selectgo` 里会主要用到，这里先看一下。结合剩下这段代码，包括 `OAS` 赋值节点 `r` 的创建，相当于让编译器插入 `selv := [ncase]scase{}` 这么个语句

而后面则类似地创建了对应 `uint16` 数组的节点用来放 order ， 也就是相当于让编译器插入 `order := [ncase * 2]uint16` ，注意这里创建的数组元素数量 `2 * ncas` ，后面 `selectgo` 里面讨论 lockOrder 与 pollOrder 的时候会有呼应。

初始化需要的变量之后，忽略竞争检测的代码，编译器就要生成把目前 select 这些 cases 塞到 `selv` 数组里的代码了，后面的这些代码就是做这个的：

### 1.4.2 准备 scase 数组

```go
	// register cases
	for _, cas := range cases.Slice() {
		setlineno(cas)

		init = append(init, cas.Ninit.Slice()...)
		cas.Ninit.Set(nil)

		n := cas.Left
		if n == nil { // default:
			continue
		}

		var i int
		var c, elem *Node
		switch n.Op {
		default:
			Fatalf("select %v", n.Op)
		case OSEND:
			i = nsends
			nsends++
			c = n.Left
			elem = n.Right
		case OSELRECV, OSELRECV2:
			nrecvs++
			i = ncas - nrecvs
			c = n.Right.Left
			elem = n.Left
		}

		casorder[i] = cas

		setField := func(f string, val *Node) {
			r := nod(OAS, nodSym(ODOT, nod(OINDEX, selv, nodintconst(int64(i))), lookup(f)), val)
			r = typecheck(r, ctxStmt)
			init = append(init, r)
		}

		c = convnop(c, types.Types[TUNSAFEPTR])
		setField("c", c)
		if elem != nil {
			elem = convnop(elem, types.Types[TUNSAFEPTR])
			setField("elem", elem)
		}

		// TODO(mdempsky): There should be a cleaner way to
		// handle this.
		if flag_race {
			r = mkcall("selectsetpc", nil, nil, nod(OADDR, nod(OINDEX, pcs, nodintconst(int64(i))), nil))
			init = append(init, r)
		}
	}
	if nsends+nrecvs != ncas {
		Fatalf("walkselectcases: miscount: %v + %v != %v", nsends, nrecvs, ncas)
	}
```
（`src/cmd/compile/internal/gc/select.go`, 265-318 行）

这段代码展开来也不复杂，整体上就是遍历这些 cases ，根据他们的类型（`OSEND`, `OSELRECV`, `OSELRECV2`）从他们的左操作数、右操作数里找到 Channel 以及对应的变量，放到 `scase` 的 `c` 与 `elem` 的字段里。转换后的代码有点像：

```go
for i, cas := range cases {
	selv[i].c = unsafe.Pointer(...) // n := cas.Left,  根据 n.Op, 可能是 n.Left 或 n.Right.Left
	selv[i].elem = unsafe.Pointer(...) // 根据 n.Op, 可能是 n.Right 或 n.Left
}
```
和原书中的有些出入，但理解这段代码的核心作用就行，就是将 select 的这些 cases 转换放到 `selv` 这个 `scase` 数组里，为后续调用 `selectgo` 做准备。

**这里值得注意的是循环过程中对于 `casorder`、`nsends`、`nrecvs`、`i` 的处理，这里对于 send 发送的通道的 i 计数是递增的（i = nsends）， 而对于 recv 接收通道的  i 计数是递减的 （i = ncas - nrecvs），而 selv 根据 i 下标来迭代创建 scase ， 这么做事实上在 selv 的数组中对于通道的类型进行了分类： 前 nsends 个是发送通道，后 nrecvs 个是接收通道。这个分批在后面 selectgo 会用到**

### 1.4.3 调用 selectgo 函数

接着就是 `walkselectcases` 生成调用 `selectgo` 的代码了：

```go
	// run the select
	lineno = sellineno
	chosen := temp(types.Types[TINT])
	recvOK := temp(types.Types[TBOOL])
	r = nod(OAS2, nil, nil)
	r.List.Set2(chosen, recvOK)
	fn := syslook("selectgo")
	r.Rlist.Set1(mkcall1(fn, fn.Type.Results(), nil, bytePtrToIndex(selv, 0), bytePtrToIndex(order, 0), pc0, nodintconst(int64(nsends)), nodintconst(int64(nrecvs)), nodbool(dflt == nil)))
	r = typecheck(r, ctxStmt)
	init = append(init, r)
```
（`src/cmd/compile/internal/gc/select.go`, 320-329 行）

这部分代码就是编译器生成调用 `selectgo` 的代码，这里创建了声明临时变量 `chosen` 与 `recvOK` 的语句，然后通过一个 `OAS2` 赋值语句节点 `r` 生成类似这样的调用语句：

```go
chosen, recvOK := selectgo(selv, orders, pc0, nsends, nrecvs, dflt == nil)
```

`dflt == nil` 代表是否有 `default` 代码块，也对应整个 select 语句会不会阻塞（如果有 `default` 就不会阻塞）。

这里的 pc0 是竞争检测相关的，可以忽略。 关于 `selectgo` 函数的分析后面会单开一章重点讲，现在只需要知道通过传入 selv 、orders ， `selectgo` 会根据传入的 case 进行不同的通道操作处理（就是处理对应 `send/recv` 的通道操作），并且会确定这次选中的需要执行的 case ，并作为返回值给到 `chosen`。 而另一个返回值 `recvOK` ，如果选择的 case 是通道接收操作（recv），那么它就代表这次接收是否成功。

下面我们先继续下去，把整个 `walkselectcases` 函数跑完。

### 1.4.4 dispatch cases

编译器在生成调用 `selectgo` 语句之后，又生成了清理销毁临时变量 `selv` 与 `orders` 的语句：

```go
	// selv and order are no longer alive after selectgo.
	init = append(init, nod(OVARKILL, selv, nil))
	init = append(init, nod(OVARKILL, order, nil))
	if flag_race {
		init = append(init, nod(OVARKILL, pcs, nil))
	}
```
（`src/cmd/compile/internal/gc/select.go`, 331-336 行）

然后，我们通过 `selectgo` 获得了要执行的 case 的序号 （`chosen`），我们编译器要生成一系列的条件判断代码：当我们 case 的序号与这次生成的 `chosen` 相同时（因为 `chosen` 的序号有可能是随机的），就生成对应 case 代码块的代码:

```go
	// dispatch cases
	dispatch := func(cond, cas *Node) {
		cond = typecheck(cond, ctxExpr)
		cond = defaultlit(cond, nil)

		r := nod(OIF, cond, nil)

		if n := cas.Left; n != nil && n.Op == OSELRECV2 {
			x := nod(OAS, n.List.First(), recvOK)
			x = typecheck(x, ctxStmt)
			r.Nbody.Append(x)
		}

		r.Nbody.AppendNodes(&cas.Nbody)
		r.Nbody.Append(nod(OBREAK, nil, nil))
		init = append(init, r)
	}

	if dflt != nil {
		setlineno(dflt)
		dispatch(nod(OLT, chosen, nodintconst(0)), dflt)
	}
	for i, cas := range casorder {
		setlineno(cas)
		dispatch(nod(OEQ, chosen, nodintconst(int64(i))), cas)
	}
```
（`src/cmd/compile/internal/gc/select.go`, 338-363 行）

我们先来看 `dispatch` 这个匿名闭包函数，它的作用就是传入的 `cond` 生成一个 if 判断语句，然后如果传入的 case 类型是 `OSELRECV2`，就会把 `selectgo` 返回的 `recvOK` 给到 `n.List.First()` ，相当于赋值了通道接收的 ok 结果。

然后这个传入的 `cond` ，看后续的代码就是 `chosen` 与 cases 下标序号比较的判断，所以这部分生成的代码相当于：

```go
if chosen < 0 {
	// default 部分代码
	break
}

if chosen == 1 {
	// case 1 主体代码
	break
}

if chosen == 2 {
	// case2 主体代码
	break
}

// ...
```

至此，`walkselectcases` 代码就都执行完毕了，函数会把生成的语句作为返回值返回出去。

现在我们来看 `runtime.selectgo`

# 2. selectgo

我们可以在 `src/runtime/select.go` 找到函数 `selectgo`：

```go
// selectgo implements the select statement.
//
// cas0 points to an array of type [ncases]scase, and order0 points to
// an array of type [2*ncases]uint16 where ncases must be <= 65536.
// Both reside on the goroutine's stack (regardless of any escaping in
// selectgo).
//
// For race detector builds, pc0 points to an array of type
// [ncases]uintptr (also on the stack); for other builds, it's set to
// nil.
//
// selectgo returns the index of the chosen scase, which matches the
// ordinal position of its respective select{recv,send,default} call.
// Also, if the chosen scase was a receive operation, it reports whether
// a value was received.
func selectgo(cas0 *scase, order0 *uint16, pc0 *uintptr, nsends, nrecvs int, block bool) (int, bool) 
```

`selectgo` 的代码就更长了（121-582），我们还是一部分一部分地来看

## 2.1 初始化准备

函数一开始，就是初始化 `scases`、 `pollOrder`、`lockOrder` 等变量

```go
	// NOTE: In order to maintain a lean stack size, the number of scases
	// is capped at 65536.
	cas1 := (*[1 << 16]scase)(unsafe.Pointer(cas0))
	order1 := (*[1 << 17]uint16)(unsafe.Pointer(order0))

	ncases := nsends + nrecvs
	scases := cas1[:ncases:ncases]
	pollorder := order1[:ncases:ncases]
	lockorder := order1[ncases:][:ncases:ncases]
```
（ `src/runtime/select.go` ， 126-134 行）

`cas1`、`order` 是基于传入 `cas0` 、`order0` 参数进行类型转换后的变量，而后面四个变量是比较重要后面函数会主要用到的：

+ ncases - 代表 cases 总数
+ scases - 对应 `walkselectcases` 传入的 `selv`， 注意这里的 `[:ncases:ncases]` , 我一开始没整明白，后来回头看 Go 语法才想起来这代表创建的 slice 取前 `ncases` 个元素，同时对应的容量（`cap`）设为 `ncases`
+ pollOrder - 从 `orders` 取 `ncases` 个作为 slice 给自己用
+ lockOrder - 从 `orders` 取后半 `ncases` 个作为 slice 给自己用，所以这里也解释了在 `walkselectcases` 里 `orders` 初始化创建了 `ncases * 2` 个元素。

忽略竞争检测、profile 的有关代码，下面就是在 pollOrder 生成随机的轮询 case 的次序。

## 2.2 pollOrder 排序

```go
	// The compiler rewrites selects that statically have
	// only 0 or 1 cases plus default into simpler constructs.
	// The only way we can end up with such small sel.ncase
	// values here is for a larger select in which most channels
	// have been nilled out. The general code handles those
	// cases correctly, and they are rare enough not to bother
	// optimizing (and needing to test).

	// generate permuted order
	norder := 0
	for i := range scases {
		cas := &scases[i]

		// Omit cases without channels from the poll and lock orders.
		if cas.c == nil {
			cas.elem = nil // allow GC
			continue
		}

		j := fastrandn(uint32(norder + 1))
		pollorder[norder] = pollorder[j]
		pollorder[j] = uint16(i)
		norder++
	}
	pollorder = pollorder[:norder]
	lockorder = lockorder[:norder]
```
（ `src/runtime/select.go` ， 157-182 行）

这部分代码主要就是在 `pollorder` 生成后续轮询 case 的顺序，整个过程会先看 `scases` 里哪些 channel 是 nil 的，有空通道的话函数会将这个 case 从后续操作里排除。

然后这个随机化的操作有点像是一种 `Fisher-Yates` 的 Shuffle 算法，想象 `scases` 是一个牌堆，我们每次从牌堆牌顶拿一张牌，然后我们将这张牌和我们手里的牌里随机一张牌交换位置，并把交换出来的牌放到自己那堆牌的牌底，然后这样不断拿牌洗牌，直到把 `scases` 牌堆所有的牌拿完洗完。

然后由于有一些 channel 为空的 case 被跳过了，所以 pollOrder 放入的元素可能是会比之前少的，所以通过 `pollorder = pollorder[:norder]` 与 `lockorder = lockorder[:norder]` 重新整了下 slice。


## 2.3 lockOrder 排序 （堆排序）

```go
	// sort the cases by Hchan address to get the locking order.
	// simple heap sort, to guarantee n log n time and constant stack footprint.
	for i := range lockorder {
		j := i
		// Start with the pollorder to permute cases on the same channel.
		c := scases[pollorder[i]].c
		for j > 0 && scases[lockorder[(j-1)/2]].c.sortkey() < c.sortkey() {
			k := (j - 1) / 2
			lockorder[j] = lockorder[k]
			j = k
		}
		lockorder[j] = pollorder[i]
	}
	for i := len(lockorder) - 1; i >= 0; i-- {
		o := lockorder[i]
		c := scases[o].c
		lockorder[i] = lockorder[0]
		j := 0
		for {
			k := j*2 + 1
			if k >= i {
				break
			}
			if k+1 < i && scases[lockorder[k]].c.sortkey() < scases[lockorder[k+1]].c.sortkey() {
				k++
			}
			if c.sortkey() < scases[lockorder[k]].c.sortkey() {
				lockorder[j] = lockorder[k]
				j = k
				continue
			}
			break
		}
		lockorder[j] = o
	}
```
（ `src/runtime/select.go` ， 184-227 行）

前面整理完 `pollOrder` 的随机顺序，这部分代码就主要就是确定 `lockorder` 也就是后面给 case 里的 channel 加锁解锁的顺序了，这里 `lockorder` 的顺序是基于 `scase.c` 对应通道的内存地址进行排序来确立的，而上面对应代码使用的就是堆排序。

堆排序保证 O(nlogn) 时间复杂度，并且是原地（in-place）的排序方法，也不会有递归调用产生额外的栈开销，实现也简单。这里也可以看到堆排序的时候所用的排序键就是通过 `c.sortkey` 方法调用的返回值，我们看下这个方法（`src/runtime/select.go`）：

```go
func (c *hchan) sortkey() uintptr {
	return uintptr(unsafe.Pointer(c))
}
```

确实就是返回 `c (*hchan)` 的值，也就是对应 `hchan` 结构体的内存地址。 

## 2.4 sellock 给 channel 加锁

确认了 `pollOrder` 轮询顺序与 `lockorder` 加锁顺序后， `selectgo` 就会调用 `sellock` 给 `scases` 里的通道们加锁：

```go
	// lock all the channels involved in the select
	sellock(scases, lockorder)
```
（ `src/runtime/select.go` ， 229-230 行）

我们看下 `sellock` 这个函数（`src/runtime/select.go`）：

```go
func sellock(scases []scase, lockorder []uint16) {
	var c *hchan
	for _, o := range lockorder {
		c0 := scases[o].c
		if c0 != c {
			c = c0
			lock(&c.lock)
		}
	}
}
```

`sellock` 就是按照 `lockorder` 的顺序给 `scases.c` 加锁，这里有个操作就是每次会把轮询的 `scase.c` 给到变量 `c`，用来与每个循环的新元素 `c0` 比较以判重，以避免给同个通道进行重复加锁。

锁上所有的通道之后，`selectgo` 函数开始了三次遍历，去处理 `select` 语句中的那些通道操作。

## 2.5 第一次遍历 - 查找已经在等待的通道

```go
	var (
		gp     *g
		sg     *sudog
		c      *hchan
		k      *scase
		sglist *sudog
		sgnext *sudog
		qp     unsafe.Pointer
		nextp  **sudog
	)

	// pass 1 - look for something already waiting
	var casi int
	var cas *scase
	var caseSuccess bool
	var caseReleaseTime int64 = -1
	var recvOK bool
	for _, casei := range pollorder {
		casi = int(casei)
		cas = &scases[casi]
		c = cas.c

		if casi >= nsends {
			sg = c.sendq.dequeue()
			if sg != nil {
				goto recv
			}
			if c.qcount > 0 {
				goto bufrecv
			}
			if c.closed != 0 {
				goto rclose
			}
		} else {
			if raceenabled {
				racereadpc(c.raceaddr(), casePC(casi), chansendpc)
			}
			if c.closed != 0 {
				goto sclose
			}
			sg = c.recvq.dequeue()
			if sg != nil {
				goto send
			}
			if c.qcount < c.dataqsiz {
				goto bufsend
			}
		}
	}

	
	if !block {
		selunlock(scases, lockorder)
		casi = -1
		goto retc
	}


```
（ `src/runtime/select.go` ， 232-286 行）

锁上通道之后，`selectgo` 开始了第一轮对于 scases 的遍历，这个循环的目标是找到有 goroutine 等待阻塞的通道，这样就可以进行优先对于这个通道进行操作。

我们可以看到，这一边遍历的次序是基于 `pollorder` 的，也就是我们随机生成的次序。 值得注意的是代码中的条件判断 `casi >= nsends` ，这个条件判断与我们 `walkselectcases` 函数中准备 `scase` 数组呼应，我们在生成 `scase` 数组时候就把 case 中的通道做了分类，`selv` 的前 `nsends` 个都是发送通道，而后 `nrecvs` 个是接收通道，所以当我们从 `pollorder` 摇到的下标是大于等于 `nsends` 的时候，说明这就是个接收通道，而后我们基于这个通道 `hchan` 对应字段反映的通道状态进行处理。

我们先看一下我们对于 case 里是一个接收通道的时候处理的场景

### 2.5.1 接收通道处理

第一种情况， `sg = c.sendq.dequeue()` 看看目前 `c` 的发送等待队列里有没有 `sudog`，有的话（`sg != nil`）就跳转到 recv 标签的代码，来处理通道接收的逻辑：

```go
recv:
	// can receive from sleeping sender (sg)
	recv(c, sg, cas.elem, func() { selunlock(scases, lockorder) }, 2)
	if debugSelect {
		print("syncrecv: cas0=", cas0, " c=", c, "\n")
	}
	recvOK = true
	goto retc
```
（ `src/runtime/select.go` ， 455-462 行）

核心就是拿调用 `recv` 函数处理接收通道发现 `sendq` 有 sudog 的情况，这部分的调用前提和逻辑其实和 `chanrecv` 函数里的处理几乎是一样的，在 Channel 的源码分析笔记中已经分析过这里不赘述了。

不过这里的入参 `unlockf` 对应调用了 `selunlock`，可以看下这个解锁的源码：

```go
func selunlock(scases []scase, lockorder []uint16) {
	// We must be very careful here to not touch sel after we have unlocked
	// the last lock, because sel can be freed right after the last unlock.
	// Consider the following situation.
	// First M calls runtime·park() in runtime·selectgo() passing the sel.
	// Once runtime·park() has unlocked the last lock, another M makes
	// the G that calls select runnable again and schedules it for execution.
	// When the G runs on another M, it locks all the locks and frees sel.
	// Now if the first M touches sel, it will access freed memory.
	for i := len(lockorder) - 1; i >= 0; i-- {
		c := scases[lockorder[i]].c
		if i > 0 && c == scases[lockorder[i-1]].c {
			continue // will unlock it on the next iteration
		}
		unlock(&c.lock)
	}
}
```
和 `sellock` 类似，只不过是 `lockorder` 的逆序进行解锁。

看完发送侧等待队列有 `sudog` 的情况，我们看看后续的条件 `c.qcount > 0`，也就是等待队列里没有 `sudog`，但是通道的缓冲区里有值的情况，就直接跳转到 `bufrecv` 了：

```go
bufrecv:
	// can receive from buffer
	if raceenabled {
		if cas.elem != nil {
			raceWriteObjectPC(c.elemtype, cas.elem, casePC(casi), chanrecvpc)
		}
		racereleaseacquire(chanbuf(c, c.recvx))
	}
	if msanenabled && cas.elem != nil {
		msanwrite(cas.elem, c.elemtype.size)
	}
	recvOK = true
	qp = chanbuf(c, c.recvx)
	if cas.elem != nil {
		typedmemmove(c.elemtype, cas.elem, qp)
	}
	typedmemclr(c.elemtype, qp)
	c.recvx++
	if c.recvx == c.dataqsiz {
		c.recvx = 0
	}
	c.qcount--
	selunlock(scases, lockorder)
	goto retc
```
（ `src/runtime/select.go` ， 412-435 行）

这部分处理逻辑也与 `chanrecv` 处理缓冲区数据的逻辑基本一模一样的，把缓冲区的值拿出来复制到 `cas.elem` 指向的内存地址。

而第三种情况也就是最后一种情况，看 `c.closed != 0` ，其实就是发现通道被关闭的情况，跳转到 `rclose`:

```go
rclose:
	// read at end of closed channel
	selunlock(scases, lockorder)
	recvOK = false
	if cas.elem != nil {
		typedmemclr(c.elemtype, cas.elem)
	}
	if raceenabled {
		raceacquire(c.raceaddr())
	}
	goto retc

```
（ `src/runtime/select.go` ， 464-474 行）

主要就是 `selunlock` 解锁、`recvOK = false` 以及将 `cas.elem` 指向的内存清零。 不过值得注意的是， 接收侧通道关闭这个 case 事实上是会被 select 语句处理的一种 case，也就是说**通道关闭在 select 语句中是会被响应的一种 case（响应了这个 case 这次 select 语句就不响应其他 case 了）**

### 2.5.2 发送通道处理

对于发送通道，处理逻辑是差不多的，不过发送通道首先是看的 `c.closed` 情况，如果 `c.closed != 0`，代表通道已被关闭，代码就会跳到 `sclose` 标签：

```go
sclose:
	// send on closed channel
	selunlock(scases, lockorder)
	panic(plainError("send on closed channel"))
```
（ `src/runtime/select.go` ， 496-500 行）

解锁、panic 报错，其他没啥好说的。

然后和接收通道类似，函数从接收等待队列 `c.recvq` 尝试获取 `sudog`，如果获取到了，就跳转到 `send` 标签：

```go
send:
	// can send to a sleeping receiver (sg)
	if raceenabled {
		raceReadObjectPC(c.elemtype, cas.elem, casePC(casi), chansendpc)
	}
	if msanenabled {
		msanread(cas.elem, c.elemtype.size)
	}
	send(c, sg, cas.elem, func() { selunlock(scases, lockorder) }, 2)
	if debugSelect {
		print("syncsend: cas0=", cas0, " c=", c, "\n")
	}
	goto retc
```
（ `src/runtime/select.go` ， 476-488 行）

调用 send 函数，和 `chansend` 函数处理这部分逻辑是一样的。


然后类似地，处理发送通道缓冲区有值的情况，跳转到 `bufsend`:

```go
bufsend:
	// can send to buffer
	if raceenabled {
		racereleaseacquire(chanbuf(c, c.sendx))
		raceReadObjectPC(c.elemtype, cas.elem, casePC(casi), chansendpc)
	}
	if msanenabled {
		msanread(cas.elem, c.elemtype.size)
	}
	typedmemmove(c.elemtype, chanbuf(c, c.sendx), cas.elem)
	c.sendx++
	if c.sendx == c.dataqsiz {
		c.sendx = 0
	}
	c.qcount++
	selunlock(scases, lockorder)
	goto retc
```
（`src/runtime/select.go` ， 437-453 行）

还是和 `chansend` 处理逻辑一样，不多讲了

然后可以看到这些跳转的代码块会在执行成功的差不多最后再跳转到 `retc`，可以看一样 `retc` 的代码：

```go
retc:
	if caseReleaseTime > 0 {
		blockevent(caseReleaseTime-t0, 1)
	}
	return casi, recvOK
```

返回被选中的 `casi` 与 `recvOK`。

可以看到这第一次遍历就是处理所有 case 中通道等待队列有 goroutine 、接收时缓冲区有值、发送时缓冲区有空位的情况。第一次遍历结束后，如果没有 case 被选中，也就是没有可以直接执行的 case，那么就看一眼有没有 `default` （`!block`），如果有的话就解锁，将 `casi` 设置为 -1，作为返回值退出 `selectgo` 函数。

## 2.6 第二次遍历 - 所有通道加入等待队列，休眠当前 goroutine

如果 `selectgo` 跑到这里，说明没有 case 是被选中能够直接运行处理的，那么也就是说我们 `select` 语句需要休眠，等待有 channel 准备 OK 然后唤醒我们的 goroutine。

```go
	// pass 2 - enqueue on all chans
	gp = getg()
	if gp.waiting != nil {
		throw("gp.waiting != nil")
	}
	nextp = &gp.waiting
	for _, casei := range lockorder {
		casi = int(casei)
		cas = &scases[casi]
		c = cas.c
		sg := acquireSudog()
		sg.g = gp
		sg.isSelect = true
		// No stack splits between assigning elem and enqueuing
		// sg on gp.waiting where copystack can find it.
		sg.elem = cas.elem
		sg.releasetime = 0
		if t0 != 0 {
			sg.releasetime = -1
		}
		sg.c = c
		// Construct waiting list in lock order.
		*nextp = sg
		nextp = &sg.waitlink

		if casi < nsends {
			c.sendq.enqueue(sg)
		} else {
			c.recvq.enqueue(sg)
		}
	}

	// wait for someone to wake us up
	gp.param = nil
	// Signal to anyone trying to shrink our stack that we're about
	// to park on a channel. The window between when this G's status
	// changes and when we set gp.activeStackChans is not safe for
	// stack shrinking.
	atomic.Store8(&gp.parkingOnChan, 1)
	gopark(selparkcommit, nil, waitReasonSelect, traceEvGoBlockSelect, 1)
	gp.activeStackChans = false
```
（`src/runtime/select.go` ， 288-328 行）

第二次遍历 `scases` 基于 `lockorder` ， 在循环中函数为每个 case 基于它的 channel 以及当前 goroutine 创建了对应的 `sudog` 结构体，设置的字段和 `chansend`/`chanrecv` 设置的很像，除了设置了 `isSelect` 为 true，然后将其加入到每个 case 对应 channel 的等待队列中（发送通道放到发送队列 `c.sendq`，接收通道放到接收队列 `c.revcq`）。

除了将 `sudog` 放到 channel 的等待队列中，同样的 `sudog` 还被以链表的形式按照 lockorder 的次序插入到 `gp.waiting` 上去（通过 `sudog.waitlink` 串联起来的链表），也就是当前 goroutine 的 waiting 字段，后面我们会看到为什么要这么做。

将 `gp.param` 设置为 nil，后面唤醒我们的 goroutine 会设置这个 `gp.param`。 然后和 channel 的发送接收代码一样设置好 `gp.parkingOnChan` 之后，就是调用 `gopark` 了。

这里 `selectgo` 调用 `gopark` 传的 `unlockf` 参数与 `chansend` / `chanrecv` 的 `gopark` 调用传的函数是不同的，为 `selparkcommit` ，我们看下这个函数：

```go
func selparkcommit(gp *g, _ unsafe.Pointer) bool {
	// There are unlocked sudogs that point into gp's stack. Stack
	// copying must lock the channels of those sudogs.
	// Set activeStackChans here instead of before we try parking
	// because we could self-deadlock in stack growth on a
	// channel lock.
	gp.activeStackChans = true
	// Mark that it's safe for stack shrinking to occur now,
	// because any thread acquiring this G's stack for shrinking
	// is guaranteed to observe activeStackChans after this store.
	atomic.Store8(&gp.parkingOnChan, 0)
	// Make sure we unlock after setting activeStackChans and
	// unsetting parkingOnChan. The moment we unlock any of the
	// channel locks we risk gp getting readied by a channel operation
	// and so gp could continue running before everything before the
	// unlock is visible (even to gp itself).

	// This must not access gp's stack (see gopark). In
	// particular, it must not access the *hselect. That's okay,
	// because by the time this is called, gp.waiting has all
	// channels in lock order.
	var lastc *hchan
	for sg := gp.waiting; sg != nil; sg = sg.waitlink {
		if sg.c != lastc && lastc != nil {
			// As soon as we unlock the channel, fields in
			// any sudog with that channel may change,
			// including c and waitlink. Since multiple
			// sudogs may have the same channel, we unlock
			// only after we've passed the last instance
			// of a channel.
			unlock(&lastc.lock)
		}
		lastc = sg.c
	}
	if lastc != nil {
		unlock(&lastc.lock)
	}
	return true
}
```

可以看到这个函数主要就是 select 语句里把所有上锁的 channel 进行一个解锁。调用 `selparkcommit` 的时候，我们已经跳出原先的 goroutine，是 `g0` 在调用了，所以我们把需要 `unlock` 的 channel 放在对应 `sudog` ， 再整合成一个链表放到对应 goroutine 的 `gp.waiting` 里，在 `selparkcommit` 函数里再拿出来做这些 channel 的解锁操作。

> 我这里有个困惑是，为什么 `gp.waiting` 取 channel 解锁的次序是顺序的？ 而 `selunlock` 解锁的次序是与 `lockorder` 次序相反的，我本来理解这么操作是为了防止死锁，但 `gp.waiting` unlock 的时候却没有倒序去弄。希望有老板能够解答一下。

## 2.7 第三次遍历 - 清理等待队列 sudog ，记录成功的 case

在 `gopark` 之后，后面就是 goroutine 被唤醒之后的逻辑了。 接下来的操作就是做好收尾工作，主要就是把各个 channel 里放着的 `sudog` 清理一下，然后通过我们的 `sudog` 匹配到成功的 channel ，以及对应的 case，作为我们 `selectgo` 的返回值。

我们看下这部分的源码（为了聚焦 selectgo 本身的逻辑，忽略了竞争检测、debug 等代码）：

```go
	sellock(scases, lockorder)

	gp.selectDone = 0
	sg = (*sudog)(gp.param)
	gp.param = nil

	// pass 3 - dequeue from unsuccessful chans
	// otherwise they stack up on quiet channels
	// record the successful case, if any.
	// We singly-linked up the SudoGs in lock order.
	casi = -1
	cas = nil
	caseSuccess = false
	sglist = gp.waiting
	// Clear all elem before unlinking from gp.waiting.
	for sg1 := gp.waiting; sg1 != nil; sg1 = sg1.waitlink {
		sg1.isSelect = false
		sg1.elem = nil
		sg1.c = nil
	}
	gp.waiting = nil

	for _, casei := range lockorder {
		k = &scases[casei]
		if sg == sglist {
			// sg has already been dequeued by the G that woke us up.
			casi = int(casei)
			cas = k
			caseSuccess = sglist.success
			if sglist.releasetime > 0 {
				caseReleaseTime = sglist.releasetime
			}
		} else {
			c = k.c
			if int(casei) < nsends {
				c.sendq.dequeueSudoG(sglist)
			} else {
				c.recvq.dequeueSudoG(sglist)
			}
		}
		sgnext = sglist.waitlink
		sglist.waitlink = nil
		releaseSudog(sglist)
		sglist = sgnext
	}

	if cas == nil {
		throw("selectgo: bad wakeup")
	}

	c = cas.c

	if casi < nsends {
		if !caseSuccess {
			goto sclose
		}
	} else {
		recvOK = caseSuccess
	}

	selunlock(scases, lockorder)
	goto retc
```

因为之前 gopark 的时候将所有 case 的对应 channel 都解锁了，所以一开始我们就调用 `sellock` 把所有 channel 锁上。 锁了之后我们定义了一些变量，然后把 `gp.waiting` 链表给清理了（但对应 sudog 都还在，只不过给到了 `sglist` 变量）

这里有个语句是 `sg = (*sudog)(gp.param)` ，这个 sg 是从 `gp.param` 拿来的值，我们回头去看下通道直接发送 `send` 与通道直接接收 `recv` 的代码，这里面就能看到在调用 `goready` 之前，我们会把等待队列里拿出来的 `sudog` 结构体，赋值给对应的 goroutine 的 `gp.param` 字段，所以这个 `sg` 拿到的就是这次唤醒我们的 channel 对应的 `sudog`。

接着我们依照 `lockorder` 的次序开始循环，通过 `sg` 在 `sglist` 里寻找这次唤醒我们的 channel 对应的 `sudog`，并得到了对应 case 的序号（`casi`）作为返回值。在这个过程中，如果我们发现 `sg` 与 `sglist` 迭代的 `sudog` 不相等，我们会把 `sglist` 对应的 `sudog` 从通道对应的等待队列中拿出来（`dequeueSudoG`），因为我们已经有成功了的 case，其他 case 对应 channel 之前放的 `sudog`，都可以处理掉了。

我们看一眼 `dequeueSudoG` 方法的源码：

```go

func (q *waitq) dequeueSudoG(sgp *sudog) {
	x := sgp.prev
	y := sgp.next
	if x != nil {
		if y != nil {
			// middle of queue
			x.next = y
			y.prev = x
			sgp.next = nil
			sgp.prev = nil
			return
		}
		// end of queue
		x.next = nil
		q.last = x
		sgp.prev = nil
		return
	}
	if y != nil {
		// start of queue
		y.prev = nil
		q.first = y
		sgp.next = nil
		return
	}

	// x==y==nil. Either sgp is the only element in the queue,
	// or it has already been removed. Use q.first to disambiguate.
	if q.first == sgp {
		q.first = nil
		q.last = nil
	}
}
```
其实就是在链表里找到并删除对应的 `sgp` 指向的 `sudog`。 其他没啥好讲的。

不过这里要专门提一下 `waitq` 的 `dequeue` 方法，有一段代码其实是和 `selectgo` 的这部分逻辑是相关的：

```go
func (q *waitq) dequeue() *sudog {
	for {
		sgp := q.first
		if sgp == nil {
			return nil
		}
		y := sgp.next
		if y == nil {
			q.first = nil
			q.last = nil
		} else {
			y.prev = nil
			q.first = y
			sgp.next = nil // mark as removed (see dequeueSudog)
		}

		// if a goroutine was put on this queue because of a
		// select, there is a small window between the goroutine
		// being woken up by a different case and it grabbing the
		// channel locks. Once it has the lock
		// it removes itself from the queue, so we won't see it after that.
		// We use a flag in the G struct to tell us when someone
		// else has won the race to signal this goroutine but the goroutine
		// hasn't removed itself from the queue yet.
		// 注意这个语句
		if sgp.isSelect && !atomic.Cas(&sgp.g.selectDone, 0, 1) {
			continue
		}

		return sgp
	}
}
```

注意这个`sgp.isSelect && !atomic.Cas(&sgp.g.selectDone, 0, 1)` 条件语句，这里的 `isSelect` 代表 channel 是在 select 语句中的，而 `sgp.g.selectDone` ，哪个 channel 先成功出队列就会通过 CAS 设置这个变量，而慢一步的 channel 就不会从 `dequeue` 里出列返回了。 这么做是因为在 select 等待 case 里的 channel 过程中，可能短时间内有不止一个 channel ok 了，但我们的 select 一次只处理一个 case，所以通过 `g.selectDone` 让这些 channel 来竞争，只有先设置这个变量的才能拿到 `sudog`，做后面的操作。

后面就是设置一些返回值，考虑通道关闭等情况，以及解锁操作的语句了。

# 3. 总结

至此，我们围绕 `select` 语句介绍了它在编译器生成代码的逻辑，以及对应 `runtime.selectgo` 处理 case 中 channel 的情况。整体在了解过 channel 源码之后还是让人觉得比较容易理解的。在阅读方法对 channel 相关函数（`chansend`/`chanrecv`/`recv`/`send`）以及结构体（`sudog`/`hchan`）有疑问的老板可以参考[之前写的文章](https://grzhan.tech/2024/06/03/GoChannelNotes/)。

后面我应该会集中精力梳理 Go 调度器的部分，力求把整体 goroutine 的生命周期、各种 goroutine 、网络轮训器、计时器等场景都在调度器中一次性梳理清楚。

这篇文章更多是自己对于 Golang Select 源码的学习与梳理，可读性比较差，里面可能存在较多错误，如果有老板觉得有问题请及时反馈，多谢！