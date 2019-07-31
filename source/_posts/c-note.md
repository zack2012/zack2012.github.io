---
title: c++笔记
mathjax: false
date: 2019-07-27 14:57:44
categories: 编程语言
tags: 
    - c++
---

1. `public`, `protected` , `private` 三种继承的区别:  

    Public inheritance
    |Access specifier in base class|Access specifier when inherited publicly|  
    |-|-|
    |Public|Public|
    |Protected|Protected|
    |Private|Inaccessible|

    Protected inheritance  
    |Access specifier in base class|Access specifier when inherited protectedly|
    |-|-|
    |Public|Protected|
    |Protected|Protected|
    |Private|Inaccessible|

    Private inheritance
    |Access specifier in base class|Access specifier when inherited protectedly|
    |-|-|
    |Public|Protected|
    |Private|Inaccessible|
    |Protected|Protected|



2. 使用`struct`, `class`声明类的区别:  
使用`struct`，实例变量默认的Access level是`public`，在继承别的类时，默认是`public`继承. 而`class`都是`private`，例如：  
    ```c++
    struct BS {
        int i; // public 
    };
    struct DS : /*public*/ Base {};

    class BC {
        int i; // private
    };
    class DC : /*private*/ BC {};
    ```

3. uniform initialization与普通初始化方法的区别：  
    - 使用模版编程时，uniform initialization有时会有让人意外，例如：  
    ```cpp
    // 判断T是否有默认构造函数
    template <typename T>
    struct IsDefaultConstructibleT {
    private:
        template <typename U, typename = decltype(U{})>
        static char test(void *);
        
        template<typename>
        static long test(...);
    public:
        static constexpr bool value = std::is_same_v<decltype(test<T>(nullptr)), char>;
    };

    struct S {
        S() = delete;
    };

    IsDefaultConstructibleT<S>::value // true! 不是false

    ```
    如果使用`decltype(U())`就能得到正确的结果，这是因为`S{}`走的是aggregate initialization，不同的c++版本要求不一样(c++20好像没这问题，待验证)。

4. One-Definition Rule(ODR): Define noninline functions or objects exactly once across all files, and define classes, inline functions, and inline variables at most once per translation unit, making sure that all definitions for the same entity are identical.

5. 在整个程序里只能定义一次的有：  
    - Noninline functions and noninline member functions (including full specializations
of function templates)  

    - Noninline variables (essentially, variables declared in a namespace scope or in the
global scope, and without the `static` specifier)  
    - Noninline static data members

6. C++11 Value Category
    - lvalue: An lvalue is a glvalue that is not an xvalue. string literal is lvalue.

    - prvalue: A prvalue is an expression whose evaluation initializes an object or a bit-field, or
computes the value of the operand of an operator.

    - xvalue: An xvalue is a glvalue designating an object or bit-field whose resources can be
reused (usually because it is about to “expire”—the “x” in xvalue originally came
from “eXpiring value”).

    - glvalue: A glvalue is an expression whose evaluation determines the identity of an object,
bit-field, or function (i.e., an entity that has storage).

    - rvalue: An rvalue is an expression that is either a prvalue or an xvalue.

7. use `decltype((x))` to check expression value category:
    - type if x is a prvalue
    - type& if x is an lvalue
    - type&& if x is an xvalue