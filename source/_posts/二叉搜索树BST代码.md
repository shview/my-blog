---
abbrlink: 二叉搜索树BST代码与思路
categories:
- - 编程
date: '2026-03-13T11:04:33.663321+08:00'
tags:
- 学习
- C++
- 编程
title: 二叉搜索树BST代码
updated: '2026-03-13T11:04:35.504+08:00'
---
注意：非动态大小

题目

[P5076 【深基16.例7】普通二叉树（简化版） - 洛谷](https://www.luogu.com.cn/problem/P5076)

```
#include <iostream>

using namespace std;

const int MAXN = 10005;
const int INF = 2147483647;

struct Node {
    int val;      // 节点存储的数值
    int ls, rs;   // 左右子节点的下标
    int size;     // 以该节点为根的子树包含的节点总数
    int cnt;      // 当前数值出现的次数
} tree[MAXN];

int root = 0, node_count = 0;

// 更新节点的 size 属性
void update(int u) {
    tree[u].size = tree[u].cnt + tree[tree[u].ls].size + tree[tree[u].rs].size;

}

// 插入操作
void insert(int& u, int x) {
    if (!u) {
        u = ++node_count;
        tree[u].val = x;
        tree[u].ls = tree[u].rs = 0;
        tree[u].size = tree[u].cnt = 1;
        return;
    }

    if (x > tree[u].val) insert(tree[u].rs, x);
    else if (x < tree[u].val) insert(tree[u].ls, x);
    else if (x == tree[u].val) tree[u].cnt++;
    update(u);

}

// 操作 1: 查询 x 的排名
int get_rank(int u, int x) {
    if (!u) return 1;
    if (x == tree[u].val) return tree[tree[u].ls].size + 1;
    if (x < tree[u].val) return get_rank(tree[u].ls, x);
    return tree[tree[u].ls].size + tree[u].cnt + get_rank(tree[u].rs, x);

}

// 操作 2: 查询排名为 x 的数
int get_val(int u, int x) {
    if (!u) return INF;
    int left_size = tree[tree[u].ls].size;
    if (x <= left_size) {
        return get_val(tree[u].ls, x);
    }
    if (x <= left_size + tree[u].cnt) {
        return tree[u].val;
    }
    return get_val(tree[u].rs, x - (left_size + tree[u].cnt));
}

// 操作 3: 求前驱
int get_pre(int u, int x) {
    int ans = -INF;
    while (u) {
        if (tree[u].val < x) {
            ans = tree[u].val;
            u = tree[u].rs;
        }
        else {
            u = tree[u].ls;
        }
    }
    return ans;
}

// 操作 4: 求后继
int get_next(int u, int x) {
    int ans = INF;
    while (u) {
        if (tree[u].val > x) {
            ans = tree[u].val;
            u = tree[u].ls;
        }
        else {
            u = tree[u].rs;
        }
    }

    return ans;
}

int main() {
    ios::sync_with_stdio(false); // 加速 I/O
    cin.tie(0);

    int q;
    cin >> q;
    while (q--) {
        int op, x;
        cin >> op >> x;
        switch (op) {
        case 1: cout << get_rank(root, x) << "\n"; break;
        case 2: cout << get_val(root, x) << "\n"; break;
        case 3: cout << get_pre(root, x) << "\n"; break;
        case 4: cout << get_next(root, x) << "\n"; break;
        case 5: insert(root, x); break;
        }
    }
    return 0;
}
```
