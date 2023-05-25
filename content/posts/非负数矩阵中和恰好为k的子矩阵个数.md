---
title: '非负数矩阵中和恰好为k的子矩阵个数'
author: 'Ghjattu'
date: '2022-11-22'
lastmod: '2022-11-22'
slug: 'the-number-of-submatrices-whose-sum-is-k-in-nonnegative-matrix'
categories: ["XCPC"]
tags: ['Prefix Sum', 'Two Pointers']
---



## 例题链接

[**Moonlight over the Lotus Pond**](https://acm.ecnu.edu.cn/contest/576/problem/C/)

## 思路

不妨设 $n\le m$ ，枚举子矩阵的上下边界 $i,j$ ，对于给定的 $i$ 和 $j$ ，问题转化为在一段长度为 $m$ 的序列中计算和为 $k$ 的子区间个数（想象把第 $i$ 行到第 $j$ 行压缩成一行），由于题目规定**矩阵中所有元素非负**，因此可以用二维前缀和+双指针解决。

若 $n>m$，把矩阵转置即可。

## 代码

```cpp
#include "iostream"
#include "vector"
#include "string"
#include "utility"
#include "queue"
#include "set"
#include "map"
#include "algorithm"
#include "cstring"
#include "stack"
#include "cmath"
#include "numeric"

using namespace std;

typedef long long ll;
const int N = 1e5 + 7;
const ll INF = 0x3f3f3f3f3f3f3f3f;
const ll MOD = 1e9 + 7;


ll cal(int a, int b, int c, int d, vector<vector<ll>> &sum) {
    return sum[c][d] - sum[c][b - 1] - sum[a - 1][d] + sum[a - 1][b - 1];
}

int main() {
#ifndef ONLINE_JUDGE
    freopen("in.txt", "r", stdin);
#endif
    ios::sync_with_stdio(false);
    cin.tie(0);

    int t;
    cin >> t;
    while (t--) {
        int n, m;
        cin >> n >> m;
        vector<vector<ll>> mat(min(n, m) + 1, vector<ll>(max(n, m) + 1));
        vector<vector<ll>> sum(min(n, m) + 1, vector<ll>(max(n, m) + 1, 0));
        for (int i = 1; i <= n; i++)
            for (int j = 1; j <= m; j++) {
                if (n <= m) cin >> mat[i][j];
                else cin >> mat[j][i];
            }

        if (n > m) swap(n, m);
        for (int i = 1; i <= n; i++)
            for (int j = 1; j <= m; j++)
                sum[i][j] = sum[i - 1][j] + sum[i][j - 1] - sum[i - 1][j - 1] + mat[i][j];

        ll ans = 0, k;
        cin >> k;
        for (int i = 1; i <= n; i++) {
            for (int j = i; j <= n; j++) {
                int l = 1, r = 1;
                for (int c = 1; c <= m; c++) {
                    l = max(l, c);
                    r = max(r, c);
                    while (r <= m && cal(i, c, j, r, sum) <= k) r++;
                    while (l <= m && cal(i, c, j, l, sum) < k) l++;
                    ans += r - l;
                }
//                // 下面的写法也可
//                int l = 0, r = 0;
//                for (int c = 1; c <= m; c++) {
//                    while (r < c && cal(i, r + 1, j, c, sum) >= k) r++;
//                    while (l < c && cal(i, l + 1, j, c, sum) > k) l++;
//                    ans += r - l;
//                }
            }
        }
        cout << ans << "\n";
    }

    return 0;
}
```

