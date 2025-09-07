---
title: "数据对齐"
date: 2025-06-07
draft: false
tags:
    - learning record
math: true
---
提问：下面这个结构体在内存中占多少个字节？
```C
struct example{
	char a;
	int b;
}
```
答案是$8$个字节。某些同学可能会以为是$5$个字节：$char (1) + int (4) = 5$。实际上，因为硬件有数据对齐的要求，在编译时，编译器会做padding操作，实际大小是：$char (1) + padding(3) + int (4) = 8$。

## 什么是数据对齐

数据对齐是CPU对基本数据类型的合法地址作出的限制：对字节大小为$K$的类型对象，它对应的内存的起始地址应当是K的整数倍。对 x86-64 硬件，无论是否对齐都能正确工作，不过Intel还是建议对齐以提高内存系统的性能。CPU做出这个限制的优势有两个：

1. 简化了处理器和内存系统之间的接口的硬件设计。
2. 提高内存的访问效率。CPU对内存的读写是以块为单位的，假设CPU每次访问$8$字节大小的数据块。对一个$8$字节大小的类型对象，如果它的起始地址是8字节的整数倍，CPU对它的处理只需要一次内存访问。否则，可能需要两次内存访问，因为数据对象可能被存放在两个$8$字节的内存块中。

因为数据对齐的限制，对用户自定义的结构体，在编译时，编译器会做填充操作。更具体的说，编译器会在结构体的字段之间填充数据为$0$的字节，保证结构体中$K$字节大小的元素相对结构体起始地址的偏移量是$K$的整数倍。这个数据为$0$的字节被称之为padding。

## 字段之间的填充
对开头提到的结构体，假设结构体的起始地址为$0$，编译器为保证变量$b$的偏移量为$4$，在$a$和$b$之间填充$3$byte的padding，结构体的实际的大小是 $char(1) + padding(3) + int (4) = 8 byte$。

```C
struct example{
	char a; // 1 byte
		// 3 byte padding
	int b;  // 4 byte
} 
```

这是一种情况：为了保证结构体中的字段满足对齐要求，需要在字段之间插入padding。
## 结构体末尾的填充
考虑下面这个结构体
```C
struct example_1{
	int a; // 4 byte
	char b; // 1 byte
}
```

变量`a`是$4$字节大小，`b`是$1$字节大小。对单个结构体，它里面的字段是满足对齐限制的。但是当我们声明一个`example_1`类型的数组的时候会发现，对数组中的结构体，它里面的字段是无论如何都无法满足对齐要求的。

为处理这种情况，编译器会在这个结构体的末尾填充padding，保证结构体的大小是它里面元素的最大类型大小的整数倍。
```C
struct example_1{
	int a;  // 4 byte
	char b; // 1 byte
		// 3 byte padding 保证结构大小是最大类型 int 大小的整数倍
}
```


在C/C++中，可以用预处理指令`#pragma pack(n)` 修改编译器的默认对齐行为，n值要求是2的幂次，可以取1、2、4、8、16。

在VS2022中，在`#pragma pack(n)`之后的结构体定义，对K字节大小的类型对象的地址限制，取n的倍数或者K的倍数两者中的较小者。

## 总结

为了保证数据对齐，编译器会做填充操作，影响用户自定义类型的实际大小。

填充有两个原则：
1. 在结构体的字段之间填充，保证元素相对结构体起始地址的偏移量是其大小的整数倍。
2. 在结构体的尾部填充，保证结构体的大小是它其中最大元素类型的整数倍。

使用预处理指令 `#pragma pack(n)` 可以修改编译器的默认对齐行为。
## 参考
1. CSAPP：[深入理解计算机系统 (原书第3版) (豆瓣)](https://book.douban.com/subject/26912767/)
2. VS2022 文档： [pack pragma | Micro）soft Learn](https://learn.microsoft.com/en-us/cpp/preprocessor/pack?view=msvc-170)
3. GCC 文档： [Structure-Packing Pragmas - Using the GNU Compiler Collection (GCC)](https://gcc.gnu.org/onlinedocs/gcc-4.4.4/gcc/Structure_002dPacking-Pragmas.html)
