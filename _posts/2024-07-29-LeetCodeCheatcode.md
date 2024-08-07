---
title: 力扣常用代码
description: 剪短的描述
author: momo
categories:
- 算法
- LeetCode
tags:
- 算法
- LeetCode
layout: post
pin: false
math: true
date: 2024-07-29 12:30 +0800
---
## LeetCode 常用代码

### 解绑老爷爷实现

```c++
auto _ = [](){
  ios::sync_with_stdio(false);
  cin.tie(0);
  cout.tie(0);
  return true;
}();
```

### 打印数组时使用


```c++
template<typename T>
ostream &operator<<(ostream &os, vector<T> const &arr){
    os << '[';
    for(auto i = arr.begin(); i != arr.end(); i++){
        os << *i;
        if(i != arr.end() - 1)
            os << ",";
    }
    os << "]";
    return os;
}
```

或者使用这个，可以打印所有可迭代对象：
```c++
// #include <iterator>         // std::begin std::end
// #include <type_traits>      // std::is_same_v
namespace details {
using std::begin;
using std::end;
template <typename T, typename = void> 
struct is_iterable : std::false_type {};

template <typename T>
struct is_iterable<T, std::void_t<
                    decltype(begin(std::declval<T>())),
                    decltype(end(std::declval<T>()))>
>: std::true_type {};

template <typename T> 
constexpr bool is_iterable_v = is_iterable<T>::value;
} // namespace details

template <typename Container>
  requires(details::is_iterable_v<Container> && 
           !std::is_same_v<Container, std::string>)
std::ostream &operator<<(std::ostream &os, Container const &arr) {
  os << '[';
  for (auto i = arr.begin(); i != arr.end(); i++) {
    os << *i;
    if (i != arr.end() - 1)
      os << ",";
  }
  os << "]";
  return os;
}
```