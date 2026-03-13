---
abbrlink: 二叉/图 Floyd 算法与例题+DFS+DP
categories:
- - 编程
date: '2026-03-13T11:37:21.585223+08:00'
excerpt: 二叉/图 Floyd 算法与例题+DFS+DP
tags:
- 学习
- C++
- 编程
title: 二叉/图 Floyd 算法+DFS+DP
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

可能有一个问题是为什么DP仅用如此简单的判断方法v != fa，这是因为树是没有环的，所以只要不回头就可以。

其时间复杂度为$O(n^2)$，可处理$N \le 3000$的情况


DP算法

~~~
#include <iostream>
#include <vector>
#include <algorithm>

using namespace std;

const int MAXN = 105; // 题目规模 N=100，这里开大一点
struct Node {
    int w, l, r;
} nodes[MAXN];

vector<int> adj[MAXN];
long long sz[MAXN];      // 即size,存储每个子树的人口总数
long long ans[MAXN];     // 存储每个点作为医院时的总距离和
long long total_weight = 0;

// 第一次 DFS：计算每个子树的人口 sz[u] 
// 同时先算出以 1 号点为根时的总代价 ans[1]
void dfs1(int u, int fa, int depth) {
    sz[u] = nodes[u].w;
    ans[1] += (long long)nodes[u].w * depth;

    for (int v : adj[u]) {
        if (v == fa) continue;
        dfs1(v, u, depth + 1);
        sz[u] += sz[v]; // 累加子树人口
    }
}

// 第二次 DFS：换根逻辑
void dfs2(int u, int fa) {
    for (int v : adj[u]) {
        if (v == fa) continue;
        // 换根公式推导：
        // 医院从 u 移到 v，v 子树内的人近了 1 步，v 子树外的人远了 1 步
        ans[v] = ans[u] - sz[v] + (total_weight - sz[v]);
        dfs2(v, u);
    }
}

int main() {
    int n;
    if (!(cin >> n)) return 0;

    total_weight = 0;
    for (int i = 1; i <= n; i++) {
        cin >> nodes[i].w >> nodes[i].l >> nodes[i].r;
        total_weight += nodes[i].w;
        // 建立无向图
        if (nodes[i].l) {
            adj[i].push_back(nodes[i].l);
            adj[nodes[i].l].push_back(i);
        }
        if (nodes[i].r) {
            adj[i].push_back(nodes[i].r);
            adj[nodes[i].r].push_back(i);
        }
    }

    // 1. 初始化 ans[1] 和各子树人口
    ans[1] = 0;
    dfs1(1, 0, 0);

    // 2. 通过换根 DFS 算出其他所有点的 ans
    dfs2(1, 0);

    // 3. 统计结果
    long long min_ans = -1;
    for (int i = 1; i <= n; i++) {
        if (min_ans == -1 || ans[i] < min_ans) {
            min_ans = ans[i];
        }
    }

    cout << min_ans << endl;

    return 0;
}
~~~

核心代码1

~~~
// 第一次 DFS：计算每个子树的人口 sz[u] 
// 同时先算出以 1 号点为根时的总代价 ans[1]
void dfs1(int u, int fa, int depth) {
    sz[u] = nodes[u].w;
    ans[1] += (long long)nodes[u].w * depth;

    for (int v : adj[u]) {
        if (v == fa) continue;
        dfs1(v, u, depth + 1);
        sz[u] += sz[v]; // 累加子树人口
    }
}
~~~

计算在节点u上的总代价


核心代码2


~~~
// 第二次 DFS：换根逻辑
void dfs2(int u, int fa) {
    for (int v : adj[u]) {
        if (v == fa) continue;
        // 换根公式推导：
        // 医院从 u 移到 v，v 子树内的人近了 1 步，v 子树外的人远了 1 步
        ans[v] = ans[u] - sz[v] + (total_weight - sz[v]);
        dfs2(v, u);
    }
}
~~~

我们可以把全树的居民分成两拨人：

1.原先在v的人，他们可以减少一步，故1 * (-z[v]) = -sz[v].

2.其他人，他们都需要多走一步，故1 * (total_weight - sz[v]) = total_weight - sz[v].


其时间复杂度为$O(n)$，可处理$N$更多的情况。
