---
abbrlink: 二叉/图 Floyd 算法与例题+DFS
categories:
- - 编程
date: '2026-03-13T11:37:21.585223+08:00'
excerpt: 二叉/图 Floyd 算法与例题+DFS
tags:
- 学习
- C++
- 编程
title: 二叉/图 Floyd 算法+DFS
updated: '2026-03-13T12:13:41.439+08:00'
---
例题

本题虽为二叉树，但可以转为图考虑

[P1364 医院设置 - 洛谷](https://www.luogu.com.cn/problem/P1364)

B站好视频

[图-最短路径-Floyd(弗洛伊德)算法](https://www.bilibili.com/video/BV19k4y1Q7Gj)

核心代码

~~~
// Floyd 算法求全源最短路
for (int k = 1; k <= n; k++) {
    for (int i = 1; i <= n; i++) {
        for (int j = 1; j <= n; j++) {
            if (dist[i][k] + dist[k][j] < dist[i][j]) {
                dist[i][j] = dist[i][k] + dist[k][j];
            }
        }
    }
}
~~~

理解：

k为中转站，i与j为终点与起点，比较i->j与i->k->j的大小，而k（可以理解为i'与j'）也是不断更新的，所以最终就是最小的

缺点：

其时间复杂度为$O(n^3)$，仅可处理$N \le 500$的情况

DFS算法

~~~
#include <iostream>
#include <vector>
#include <algorithm>

using namespace std;

int n;
int weight[105];
vector<int> adj[105]; // 邻接表，存储树的边（无向）
long long total_dist = 0;

// u: 当前节点, fa: 父节点（防止往回跑）, d: 距离起点的距离
void dfs(int u, int fa, int d) {
    total_dist += (long long)weight[u] * d; // 累加路程和

    for (int v : adj[u]) {
        if (v != fa) { // 只要不是回头路，就继续深搜
            dfs(v, u, d + 1);
        }
    }
}

int main() {
    cin >> n;
    for (int i = 1; i <= n; i++) {
        int w, l, r;
        cin >> w >> l >> r;
        weight[i] = w;
        if (l) {
            adj[i].push_back(l);
            adj[l].push_back(i); // 树边是双向的
        }
        if (r) {
            adj[i].push_back(r);
            adj[r].push_back(i);
        }
    }

    long long min_ans = -1;

    for (int i = 1; i <= n; i++) {
        total_dist = 0;
        dfs(i, 0, 0); // 以 i 为医院位置进行搜索

        if (min_ans == -1 || total_dist < min_ans) {
            min_ans = total_dist;
        }
    }

    cout << min_ans << endl;
    return 0;
}
~~~

核心代码

~~~
// u: 当前节点, fa: 父节点（防止往回跑）, d: 距离起点的距离
void dfs(int u, int fa, int d) {
    total_dist += (long long)weight[u] * d; // 累加路程和

    for (int v : adj[u]) {
        if (v != fa) { // 只要不是回头路，就继续深搜
            dfs(v, u, d + 1);
        }
    }
}
~~~

可能有一个问题是为什么DP仅用如此简单的判断方法v != fa，这是因为树是没有环的，所以只要不回头就可以
