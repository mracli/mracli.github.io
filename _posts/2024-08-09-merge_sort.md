---
title: merge_sort
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
date: 2024-08-09 15:47 +0800
---
merge_sort也要有自己的非递归版本

## 代码

这个版本模仿后序遍历，问题在及其占用空间，复杂度在 $O(n^2)$, 所以仅供参考，实际不会这么实现。

```c++
#include <cstdio>
#include <queue>
#include <stack>
#include <unordered_map>
#include <vector>

using namespace std;

const int N = 100010;

int tmp[N];

void merge_sort(int arr[], int l, int r){
    if(l >= r) return;
    vector<int> tmp(r - l + 1);
    int mid = l + r >> 1;
    merge_sort(arr, l, mid):
    merge_sort(arr, mid + 1, r):
    int i = l, j = mid + 1, k = 0;
    while(i <= mid && j <= r){
        if(arr[i] < arr[j]) tmp[k++] = arr[i++];
        else tmp[k++] = arr[j++];
    }
    while(i <= mid) tmp[k++] = arr[i++];
    while(j <= r) tmp[k++] = arr[j++];
    for(int i = l, k = 0; i <= r; i++, k++) arr[i] = tmp[k];
}

void merge_sort(int arr[], int l, int r){
    if(l >= r) return;
    using pii = pair<int,int>;
    stack<pii> s;
    vector<vector<bool>> st(r - l + 1, vector<bool>(r - l + 1, false));
    vector<int> tmp(r - l + 1);
    
    s.push({l, r});
    
    while(s.size()){
        auto [l, r] = s.top();
        if(l >= r) {
            s.pop();
            continue;
        }
        int mid = l + r >> 1;
        if(st[l][r]){
            // 已经计算过子区间
            int i = l, j = mid + 1, k = 0;
            while(i <= mid && j <= r){
                if(arr[i] < arr[j]) tmp[k++] = arr[i++];
                else tmp[k++] = arr[j++];
            }
            while(i <= mid) tmp[k++] = arr[i++];
            while(j <= r) tmp[k++] = arr[j++];
            for(int i = l, k = 0; i <= r; i++, k++) arr[i] = tmp[k];
            s.pop();
        }else{
            s.push({l, mid});
            s.push({mid + 1, r});
            st[l][r] = true;
        }
    }
}

int main(){
    int n;
    scanf("%d", &n);
    int arr[n];
    for (int i = 0; i < n; i++) scanf("%d", &arr[i]);
    
    merge_sort(arr, 0, n - 1);
    
    for (int i = 0; i < n; i++) printf("%d ", arr[i]);
    
    return 0;
}
```

正常的实现，从小到大迭代窗口，记得处理最右的一块不是2的整数次幂的情况。

```c++
#include <iostream>
#include <vector>

using namespace std;

const int N = 100010;
static vector<int> tmp(N);

void merge_single(vector<int> &arr, int l, int r, int mid, bool p = false){
    if(l >= r) return;
    int i = l, j = mid + 1, k = 0;
    while(i <= mid && j <= r){
        if(arr[i] < arr[j]) tmp[k++] = arr[i++];
        else tmp[k++] = arr[j++];
    }
    
    while(i <= mid) tmp[k++] = arr[i++];
    while(j <= r) tmp[k++] = arr[j++];
    for(int i = l, k = 0; i <= r; i++, k++) arr[i] = tmp[k];
}

void merge_sort(vector<int> &arr, int l, int r){
    if(l >= r) return;
    int k = 2;
    for(; k <= r - l + 1; k <<= 1){
        for(int i = 0; i <= r; i += k){
            int j = min(r, i + k - 1), mid = i + (k >> 1) - 1;
            merge_single(arr, i, j, mid);
        }
    }
    merge_single(arr, l, r, (k >> 1) - 1, true);
}

int main(){
    int n;
    cin >> n; 
    vector<int> arr(n);
    for(int i = 0; i < n; i++)
        cin >> arr[i];
    merge_sort(arr, 0, n - 1);
    for(int i = 0; i < n; i++)
        cout << arr[i] << ' ';
    cout << '\n';
    
    return 0;
}
```