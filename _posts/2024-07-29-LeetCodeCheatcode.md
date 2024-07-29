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
ostream &operator<<(ostream &os, vector<T> arr){
    os << '[';
    for(auto i = arr.begin(); i != arr.end(); i++){
        os << *i;
        if(i != arr.end() - 1)
            os << ", ";
    }
    os << "]";
    return os;
}
```
