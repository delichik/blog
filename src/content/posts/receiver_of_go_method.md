---
title: 值接收器和指针接收器
published: 2024-11-27
description: ""
tags: ["Golang", "Plan9"]
category: 开发
draft: false
---
# 值接收器和指针接收器

## 你这 interface 怎么看都有问题

### 场景

有这样一个结构体，包含一个整数字段、2个指针接收器和2个值接收器：
```go
type s struct {
	v int
}

func (t *s) fn1() {
	t.v = 1
}

func (t *s) fn2() {
	t.v = 2
}

func (t s) fn3() {
	t.v = 3
}

func (t s) fn4() {
	t.v = 4
}
```

现在尝试将这个结构体和他的指针转换到以下3种interface

#### Scene 1

```go
type i1 interface {
	fn1()
	fn2()
}
var a i1 = s{v: 0} // not OK
var a i1 = &s{v: 0} // OK
```

#### Scene 2

```go
type i2 interface {
	fn1()
	fn3()
}

var a i2 = s{v: 0} // not OK
var a i2 = &s{v: 0} // OK
```

#### Scene 3

```go
type i3 interface {
	fn3()
	fn4()
}

var a i3 = s{v: 0} // OK
var a i3 = &s{v: 0} // OK
```

根据上面场景，可以发现：
| 接收器1 | 接收器2 | 调用者 | 有效 |
| ------- | ------- | ------ | ---- |
| 指针    | 指针    | 值     | 否   |
| 指针    | 指针    | 指针   | 是   |
| 值      | 指针    | 值     | 否   |
| 值      | 指针    | 指针   | 是   |
| 值      | 值      | 值     | 是   |
| 值      | 值      | 指针   | 是   |

看起来就像是：

> 调用者是指针，实现接口时，值接收器 **会** 被当作有效的实现；
>
> 调用者是值，实现接口时指针接收器 **不会** 被当作有效的实现。

### 原因

首先我们简化一下结构体方便我们观察：

```go
type s2 struct {
	v int
}

//go:noinline
func (t *s) fn1() { // 这个是个指针接收器
	t.v = 1
}

//go:noinline
func (t s) fn2() { // 这个是个值接收器
	t.v = 2
}
```

使用`go tool compile -S ./main.go`将代码转换为plan9汇编，
我们可以看到go编译器生成了3个函数（不是2个也不是4个）：

```plan9
// func (t *s) fn1()
TEXT    main.(*s2).fn1(SB), NOSPLIT|NOFRAME|ABIInternal, $0-8
...

// func (t s) fn2()
TEXT    main.s2.fn2(SB), NOSPLIT|NOFRAME|ABIInternal, $0-8
...

// 编译器确实生成了一个 func (t *s) fn2()
TEXT    main.(*s2).fn2(SB), DUPOK|WRAPPER|ABIInternal, $24-8
...
```

所以根本的原因是：

> 值接收器会自动生成一个指针接收器，指针接收器不会生成值接收器

所以无论interface的实现函数是指针还是值接收器，只要我们无脑使用结构体的指针，就一定能达到我们想要的效果了......吗？

那肯定是不行的，毕竟IDE都会告诉我们：

> Goland:
>
> Struct s2 has methods on both value and pointer receivers. Such usage is not recommended by the Go Documentation.

那么这两种使用到底有什么不一样呢？

## 

依然是使用这个简化的结构体：

```go
type s2 struct {
	v int
}

//go:noinline
func (t *s2) fn1() { // 这个是个指针接收器
	t.v = 1
}

//go:noinline
func (t s2) fn2() { // 这个是个值接收器
	t.v = 2
}
```

再次使用`go tool compile -S ./main.go`将代码转换为plan9汇编，
然后我们再一个一个的观察

### 编译后的函数

#### func (t *s) fn1()

```
0x0000 00000    TEXT    main.(*s2).fn1(SB), NOSPLIT|NOFRAME|ABIInternal, $0-8
0x0000 00000    MOVQ    AX, main.t+8(SP)     // *t 拷贝到栈
0x0005 00005    TESTB   AL, (AX)             // 检查低位是否为 0
0x0007 00007    MOVQ    $1, (AX)             // t.v = 1
0x000e 00014    RET
```

这里的 AX 内是一个指针，可以看到，第二个 MOVQ 修改的是 AX 指向的值，
所以修改会导致原本在 AX 中的值变化

#### func (t s) fn2()

```
0x0000 00000    TEXT    main.s2.fn2(SB), NOSPLIT|NOFRAME|ABIInternal, $0-8
0x0000 00000    MOVQ    AX, main.t+8(SP)    // t 拷贝到栈
0x0005 00005    MOVQ    $2, main.t+8(SP)    // 为栈中的 t.v 赋值2
0x000e 00014    RET
```
这里的 AX 内是一个结构体（这段代码里面实际是一个 int），这里的第二个 MOVQ 修改的是 AX 拷贝到函数栈中的值，
函数结束时就会被删除，所以不会导致 AX 中的值产生变化

#### func (t *s) fn2()

从上面我们知道，这个`func (t *s) fn2()`是编译器自己生成出来的，
但是他的函数内容确实和前面...可以说完全不一样，
将其中进行栈伸缩和异常检查的代码去掉后，得到一份精简后的代码

```plan9
0x0000 00000    TEXT    main.(*s2).fn2(SB), DUPOK|WRAPPER|ABIInternal, $24-8
0x0006 00006    PUSHQ   BP
0x0007 00007    MOVQ    SP, BP
0x000a 00010    SUBQ    $16, SP                   // 准备2字节的栈空间
0x000e 00014    MOVQ    32(R14), R12
0x0017 00023    MOVQ    AX, main.t+32(SP)         // 从AX读取参数 *t
0x0027 00039    TESTB   AL, (AX)
0x0029 00041    MOVQ    (AX), AX                  // 获取 *t 指向的值
0x002c 00044    MOVQ    AX, main..autotmp_1+8(SP) // 拷贝到栈空间
0x0031 00049    CALL    main.s2.fn2(SB)           // 调用真实的函数
0x0036 00054    ADDQ    $16, SP
0x003a 00058    POPQ    BP
0x003b 00059    RET
```

在这个函数中，第四个 MOVQ 将原本 AX 中的指针变成了函数栈中的值，然后由 CALL 传给了真正的 main.s2.fn2(SB)，
所以这个中间函数实际上就是负责制造一个指针指向的值的拷贝。

翻译成go，其实就是很简单一句：

```go
func (s *s2) fn2() {
  return (*s).fn2()
}
```

### 函数的使用

现在我们使用一个main函数来使用它的method

```go
func main () {
	t1 := &s2{}
	t1.fn1()
	t1.fn2()
	t2 := s2{}
	t2.fn1()
	t2.fn2()
}
```

同样，将代码转换为plan9汇编，并将其中进行栈伸缩和异常检查的代码去掉后，
得到一份精简后的代码

```
0x0000 00000    TEXT    main.main(SB), ABIInternal, $56-0
0x0006 00006    PUSHQ   BP
0x0007 00007    MOVQ    SP, BP
0x000a 00010    SUBQ    $48, SP
// 初始化第一个 s2{}
0x000e 00014    MOVQ    $0, main..autotmp_3+24(SP)
// t1.fn1()
0x0017 00023    LEAQ    main..autotmp_3+24(SP), AX // 获取第一个 s2{} 的地址到 AX 中
0x001c 00028    MOVQ    AX, main..autotmp_2+40(SP)
0x0021 00033    TESTB   AL, (AX)
0x0023 00035    MOVQ    $0, (AX)                   // 再次赋值0（？）
0x002a 00042    MOVQ    AX, main.t1+32(SP)         // 保存 AX 内容到栈上
0x002f 00047    CALL    main.(*s2).fn1(SB)         // 调用函数
// t1.fn2()
0x0034 00052    MOVQ    main.t1+32(SP), CX         // 取出刚刚的 AX
0x0039 00057    TESTB   AL, (CX)
0x003b 00059    MOVQ    (CX), AX                   // 取出 *t 指向的内容
0x003e 00062    MOVQ    AX, main..autotmp_4+16(SP) // 保存 AX 内容到栈上
0x0043 00067    CALL    main.s2.fn2(SB)            // 调用函数
// 初始化第二个 s2{}
0x0048 00072    MOVQ    $0, main.t2+8(SP)
// t2.fn1()
0x0051 00081    LEAQ    main.t2+8(SP), AX          // 获取第二个 s2{} 的地址到 AX 中
0x0056 00086    CALL    main.(*s2).fn1(SB)         // 调用函数
// t2.fn2()
0x005b 00091    MOVQ    main.t2+8(SP), AX          // 获取第二个 s2{} 的值到 AX 中
0x0060 00096    CALL    main.s2.fn2(SB)            // 调用函数

0x0065 00101    ADDQ    $48, SP
0x0069 00105    POPQ    BP
0x006a 00106    RET
```

所以上面原本的代码翻译一下就是：

```go
func main () {
	t1 := &s2{}
	t1.fn1()
	(*t1).fn2()
	t2 := s2{}
	(&t2).fn1()
	t2.fn2()
}
```