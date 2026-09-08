---
title: 正则表达式与C++11(m[].str()待修改)
date: 2023/8/10 11:07
categories: 
- C/C++
tags:
- C/C++
---

### 1. 表达式语法

> 参考网站： 
>
> - [正则表达式 – 语法 | 菜鸟教程 (runoob.com)](https://www.runoob.com/regexp/regexp-syntax.html)
>
> - [正则表达式语言 - 快速参考 | Microsoft Learn](https://learn.microsoft.com/zh-cn/dotnet/standard/base-types/regular-expression-language-quick-reference)

http://regexr.com/

http://www.regexpal.com/ 

[:heart:regex101:heart:](https://regex101.com/)

### 2. C++11 Regex基本API

​	**注：**以下函数声明并非官方原型，仅是为了方便理解使用而产生形式。详细可以查询 [cppreference.com](https://en.cppreference.com/w/cpp/regex)

- 引用头文件`#include <regex>`

- `std::regex`

  - **s**:  字符串形式的正则表达式

  ```c++
  	// 初始化一个std::regex对象作为匹配模板	
  	std::regex(std::string s);
  ```
  
- `std::regex_match`

  - **src**: 待匹配字符串
  - **m**: 传出参数包含了匹配的结果信息，匹配的字符串也会保存在这里, 注意`std::match_result`作为基类，实际使用的时候需要调用子类`std::cmatch`或者`std::smatch`, 代表结果的储存是`char*`或者`std::string::const_iterator`。
  - **r**: 上文构建的`std::regex`对象

  ```c++
  	// 将匹配结果放在m中
  	// 返回值代表匹配是否成功
  	bool std::regex_match(std::string src, std::match_result m, std::regex r);
  ```

- `std::regex_search`

  ```c++
  	bool std::regex_match(std::string src, std::match_result m, std::regex r);
  ```

- `std::regex_iterator`

### 3. 注解 

- [cppreference.com](https://en.cppreference.com/w/cpp/regex/regex_match) 给出的`regex_match`和`regex_search`的区别

> Because `regex_match` only considers full matches, the same regex may give different matches between `regex_match` and [std::regex_search](https://en.cppreference.com/w/cpp/regex/regex_search): 

```c++
    std::regex re("Get|GetValue");
    std::cmatch m;
    std::regex_search("GetValue", m, re);  // returns true, and m[0] contains "Get"
    std::regex_match ("GetValue", m, re);  // returns true, and m[0] contains "GetValue"
    std::regex_search("GetValues", m, re); // returns true, and m[0] contains "Get"
    std::regex_match ("GetValues", m, re); // returns false
```

`regex_search` 会在字符串中查找与正则表达式模式匹配的**子串**，而 `regex_match` 则要求**整个被匹配字符串**都与正则表达式模式匹配。另外`regex_search`只成功匹配一次就会返回，意味着得到所有匹配的结果需要迭代查找。

- 如何使用匹配结果？
