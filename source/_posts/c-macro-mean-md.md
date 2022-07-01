---
title: C/C++ 中宏定义中#和##的作用
author: Jack
tags:
  - C/C++
categories:
  - C/C++
abbrlink: 57aaac13
date: 2022-06-30 14:07:08
---

#### 宏定义中#的功能
C/C++宏定义#中的功能是将其后面的宏参数进行字符串化操作，简单说就是在对它所引用的宏变量通过替换后在其左右各添加一个双引号。
<!-- more -->

#### 宏定义中##的功能
宏定义中##中的功能是在带参数的宏定义中将##前后的子串进行拼接。

#### 代码
```
#include <iostream>
#include <climits>

using namespace std;

#define STR(s) #s
#define CONS(a, b) int(a##e##b)

int main()
{
    cout << STR(hello) << endl;
    cout << CONS(2, 3) << endl;     // CONS(2, 3) -> 2e3 
    return 0;
}

// 运行结果
hello
2000
```