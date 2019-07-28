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