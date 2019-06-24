---
title: 位运算
mathjax: false
date: 2019-04-23 08:25:33
categories: 编程语言
tags: 
    - swift
    - c++
---

在平时的工作中，位运算用的并不多，但在算法面试中，位运算是一种很重要的技巧。本文介绍一些常用的位运算技巧以及使用上需要注意的地方。因为c++的位运算来自c，所以下文中关于c++位运算的描述也同样使用于c和objective-c。

<!-- more -->

## 1. 位运算符

编程语言中一般都有以下6种位运算：
- `&`(与运算)，真值表如下：  

| |0|1|
|-|-|-|
|0|0|0|
|1|0|1|

- `|`(或运算)，真值表如下：

| |0|1|
|-|-|-|
|0|0|1|
|1|1|1|

- `^`(异或运算)，相同为假，不同为真，真值表如下：  

| |0|1|
|-|-|-|
|0|0|1|
|1|1|0|

- `~`(取反)，按位取反，单目运算符。

- `<<`(左移)，将一个整数向左移动n位。  

- `>>`(右移)，将一个整数向右移动n位。

## 2. 位运算注意事项

### 2.1 位运算优先级较低

在c++(以下c++还包含c、objective-c)、swift中，位运算的优先级较低，大多数情况下位运算都需要使用()，一般ide都能给予警告，但如果不使用ide的情况下，不注意优先级可能让你花费很长的时间debug。

c++符号优先级如下([来源](http://www.enseignement.polytechnique.fr/informatique/INF478/docs/Cpp/en/cpp/language/operator_precedence.html))：

|Precedence|Operator|Description|Associativity|
|-|-|-|-|
|1|	::|	Scope resolution|	Left-to-right|
|2|	++ --<br>type() type{}<br>()<br>[]<br>.<br>->|Suffix/postfix increment and decrement<br>Function-style type cast<br>Function call<br>Array subscripting<br>Element selection by reference<br>Element selection through pointer | Left-to-right |
|3|	++ --<br>+ -<br>! ~<br>(type)<br>*<br>&<br>sizeof<br>new, new[]<br>delete, delete[]| Prefix increment and decrement<br>Unary plus and minus<br>Logical NOT and bitwise NOT<br>C-style type cast<br>Indirection (dereference)<br>Address-of<br>Size-of<br>Dynamic memory allocation<br>Dynamic memory deallocation<br>| Right-to-left |
|4| .* ->* | Pointer to member | Left-to-right |
|5|	*   /   % |	Multiplication, division, and remainder | Left-to-right |
|6|	+   - |	Addition and subtraction | Left-to-right |
|7|	<<   >> | Bitwise left shift and right shift | Left-to-right |
|8|	<   <=<br> >   >=| For relational operators < and ≤ respectively <br> For relational operators > and ≥ respectively| Left-to-right |
|9|	==   !=	| For relational = and ≠ respectively | Left-to-right | 
|10| & |	Bitwise AND | Left-to-right |
|11|	^	|Bitwise XOR (exclusive or) | Left-to-right |
|12|	&#124;	| Bitwise OR (inclusive or) | Left-to-right |
|13|	&&	| Logical AND | Left-to-right |
|14|	&#124;&#124;	| Logical OR | Left-to-right |
|15| ?:<br>=<br>+=   -=<br>*=   /=   %=<br><<=   >>=<br>&=   ^=   &#124;=<br>| Ternary conditional<br>Direct assignment (provided by default for C++ classes)<br>Assignment by sum and difference<br>Assignment by product, quotient, and remainder<br>Assignment by bitwise left shift and right shift<br>Assignment by bitwise AND, XOR, and OR<br>| Right-to-left |
|16| throw |	Throw operator (for exceptions) | Right-to-left |
|17|	,  |Comma |	Left-to-right

从上表可以看出，左移、右移(`<<,>>`)的优先级高于比较运算符(`>=,<=,==,!=`)但低于四则运算符(`+,-,*,/,%`)，而一些位运算优先级(`&,|,^`)甚至低于比较运算符，在有位运算的表达式里需要特别注意

### 2.2 左移、右移操作数类型

左移、右移位运算符对符号两边的操作数类型在不同的语言中要求是不一样的。在c++中，要求两边的操作数必须是**无符号整型**。如果操作数两边的任意数为负数，则结果是[undefined behaviour](https://zh.wikipedia.org/wiki/未定义行为)。下面代码在Clion中会给予警告：

```c++
int n = 1;
int r = 1 << n; // Clang-Tidy: use of a signed integer operand with a bitwise binary
```
上面的代码完全是合法的，正数会被自动转换为无符号整数(正数转换为无符号整数保持不变)，但如果想要消除上面的警告，最简洁的做法如下：

```c++
int n = 1;
int r = 1u << (unsigned)n;
```

在swift中，对左移、右移运算符的要求就不像c++那样必须是无符号整数了。对于无符号整数，位移运算符的规则如下：

* 根据要求的位数左移或右移
* 超出范围的位丢弃
* 移动后空出的位置用0填充

这种位移也被称为**逻辑位移**.

对于有符号整数，位移符号的右边如果是正数，则表示根据符号移动相应位数，如果是负数则反向移动，即左移变右移，右移变左移，例如：

```swift
let x = 4
print(x << -1) // 2
print(x >> -1) // 8
```

对于有符号整数，位移运算符的规则如下：

* 根据要求的位数左移或右移
* 超出范围的位丢弃
* 左移空出的位置用0填充，右移空出的位置用符号位填充，对于正数就是用0填充，对于负数则是用1填充

这种位移也被称为**算数位移**.

## 3. 常见的位运算技巧

常见的位运算技巧如下：

* 对一个数左移就是乘以2，右移就是除以2
* 可以通过`n & 1`来判断`n`是偶数还是奇数
* 异或运算有很多奇淫巧技，例如：  
    * 不用临时变量来交换两个整数：
        ```c++
        int x = 10, y = 5; 
        // Code to swap 'x' (1010) and 'y' (0101) 
        x = x ^ y; // x now becomes 15 (1111) 
        y = x ^ y; // y becomes 10 (1010) 
        x = x ^ y; // x becomes 5 (0101) 
        ```
    *