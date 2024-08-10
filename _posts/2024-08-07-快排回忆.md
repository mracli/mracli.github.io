---
title: quick_sort
description: 剪短的描述
author: momo
categories:
- 算法
- 排序
tags:
- 算法
layout: post
pin: false
math: true
date: 2024-08-07 21:03 +0800
---
很久没看过算法内容，今天逛了逛牛客，看到有要求快排非递归，就试着重写了一下，统计了下时间，大概完成花费 3 分钟 13.54 秒，相比直接使用编译器提供的栈，还是要慢一些。


## 代码

```c++
#include <iostream>
#include <cstring>
#include <queue>

using namespace std;
using pii = pair<int,int>;
const int N = 100010;

void quick_sort(int arr[], int l, int r){
    if(l >= r) return;
    int i = l - 1, j = r + 1, x = arr[l + r >> 1];
    while(i < j){
        while(arr[++i] < x);
        while(arr[--j] > x);
        if(i < j) swap(arr[i], arr[j]);
    }
    quick_sort(arr, l, j);
    quick_sort(arr, j + 1, r);
}

void quick_sort(int arr[], int l, int r){
    if(l >= r) return;
    queue<pii> st;
    st.push({l, r});
    while(st.size()){
        auto [l, r] = st.front(); st.pop();
        if(l >= r) continue;
        int i = l - 1, j = r + 1, x = arr[l + r >> 1];
        while(i < j){
            while(arr[++i] < x);
            while(arr[--j] > x);
            if(i < j) swap(arr[i], arr[j]);
        }
        st.push({l, j});
        st.push({j + 1, r});
    }
}

int main(){
    int n;
    cin >> n;
    int arr[N];
    for(int i = 0; i < n; i++) cin >> arr[i];
    quick_sort(arr, 0, n - 1);
    for(int i = 0; i < n; i++) cout <<arr[i] <<' ';
    cout << '\n';
    
    return 0;
}

```