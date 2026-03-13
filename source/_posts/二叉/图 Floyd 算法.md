---
abbrlink: 二叉/图 Floyd 算法与例题
categories:
- - 编程
date: '2026-03-13T11:37:21.585223+08:00'
excerpt: 二叉/图 Floyd 算法与例题
tags:
- 学习
- C++
- 编程
title: 二叉/图 Floyd 算法
updated: '2026-03-13T11:37:23.060+08:00'
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
