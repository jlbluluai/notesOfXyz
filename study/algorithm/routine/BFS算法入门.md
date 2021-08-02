# Table of Contents

* [BFS算法入门](#bfs算法入门)
  * [介绍](#介绍)
  * [套路框架](#套路框架)
  * [题解](#题解)
    * [二叉树的最小高度](#二叉树的最小高度)
    * [解开密码锁的最少次数](#解开密码锁的最少次数)


# BFS算法入门

## 介绍

BFS（广度优先遍历），常见场景就是从起点到终点的最近距离，例如：走迷宫最短距离


## 套路框架

```
// 计算从起点 start 到终点 target 的最近距离
int BFS(Node start, Node target) {
    Queue<Node> q; // 核⼼数据结构
    Set<Node> visited; // 避免⾛回头路

    q.offer(start); // 将起点加⼊队列
    visited.add(start);
    int step = 0; // 记录扩散的步数

    while (q not empty){
        int sz = q.size();
        /* 将当前队列中的所有节点向四周扩散 */
        for (int i = 0; i < sz; i++) {
            Node cur = q.poll();
            /* 划重点：这⾥判断是否到达终点 */
            if (cur is target)
            return step;
            /* 将 cur 的相邻节点加⼊队列 */
            for (Node x : cur.adj())
                if (x not in visited){
                    q.offer(x);
                    visited.add(x);
                }
        }
        /* 划重点：更新步数在这⾥ */
        step++;
    }
}
```


## 题解

### 二叉树的最小高度

[Leetcode第111题](https://leetcode-cn.com/problems/minimum-depth-of-binary-tree/)

该题一看没有终点？其实不是，二叉树到底的终点标志是啥，左右节点都为null，这不就明显了。而且该题还可以省略已访问集合，为什么呢，因为二叉树我们按层去扩散，节点的扩散都是单向且不可能重叠，这是树结构的优势，图还是需要的。所以该题其实理解BFS稍显简单，不过先入个门，慢慢深入。

```
public int minDepth(TreeNode root) {
    // 特殊情况处理
    if (root == null) {
        return 0;
    }

    // 初始化扩散等待队列
    Queue<TreeNode> q = new LinkedList<>();
    // 初始化已尝试集合 该题不需要 二叉树是单向的树结构 先天然不会重叠

    // 初始化初始数据 起点入队列 初始化步长为1(二叉树根节点本身就算一层)
    q.offer(root);
    int step = 1;

    // 只要q还不为空 就继续扩散
    while (!q.isEmpty()) {
        int curSize = q.size();
        // 当前队列中的节点均尝试扩散
        for (int i = 0; i < curSize; i++) {
            // 若当前的节点的左右为null 说明到底 并且遍历是按层的 所以就为最短
            TreeNode cur = q.poll();
            if (cur.left == null && cur.right == null) {
                return step;
            }

            // 左右只要不为空 入队列等待下一层的扩散
            if (cur.left != null) {
                q.offer(cur.left);
            }
            if (cur.right != null) {
                q.offer(cur.right);
            }
        }

        // 该层全部结束 还未找到答案 那必然最小深度最好也在下一层 深度就+1
        step++;
    }

    return step;
}
```


### 解开密码锁的最少次数

[Leetcode第752题](https://leetcode-cn.com/problems/open-the-lock/)


自己套框架点解法

```
public int openLock(String[] deadends, String target) {
    // 初始化扩散等待队列
    Queue<String> q = new LinkedList<>();
    // 初始化已访问集合
    Set<String> visited = new HashSet<>();

    // 起点入队列，并加入已访问集合 初始化步长为0
    q.offer("0000");
    visited.add("0000");
    int step = 0;

    Set<String> deads = Arrays.stream(deadends).collect(Collectors.toSet());
    // 开始扩散
    while (!q.isEmpty()) {
        int curSize = q.size();
        for (int i = 0; i < curSize; i++) {
            String cur = q.poll();
            // 若在deadends，直接略过
            if (deads.contains(cur)) {
                continue;
            }
            // 若命中结果 返回步数
            if (target.equals(cur)) {
                return step;
            }
            // 否则继续扩散
            for (String s : adj(cur)) {
                // 若已访问过 略过
                if (visited.contains(s)) {
                    continue;
                }
                q.offer(s);
                visited.add(s);
            }
        }
        // 步数+1
        step++;
    }

    return -1;
}


/**
 * 扩散当前节点的所有可能性
 */
private String[] adj(String cur) {
    char[] chars = cur.toCharArray();
    String[] res = new String[2 * chars.length];

    for (int i = 0; i < chars.length; i++) {
        chars[i] = plus1(chars[i]);
        res[i] = String.valueOf(chars);
        chars[i] = minus1(chars[i]);
    }

    for (int i = 0; i < chars.length; i++) {
        chars[i] = minus1(chars[i]);
        res[i + chars.length] = String.valueOf(chars);
        chars[i] = plus1(chars[i]);
    }

    return res;
}

/**
 * 数字快速加1
 */
private char plus1(char c) {
    switch (c) {
        case '0':
            return '1';
        case '1':
            return '2';
        case '2':
            return '3';
        case '3':
            return '4';
        case '4':
            return '5';
        case '5':
            return '6';
        case '6':
            return '7';
        case '7':
            return '8';
        case '8':
            return '9';
        case '9':
            return '0';
        default:
            return c;
    }
}

/**
 * 数字快速减1
 */
private char minus1(char c) {
    switch (c) {
        case '0':
            return '9';
        case '1':
            return '0';
        case '2':
            return '1';
        case '3':
            return '2';
        case '4':
            return '3';
        case '5':
            return '4';
        case '6':
            return '5';
        case '7':
            return '6';
        case '8':
            return '7';
        case '9':
            return '8';
        default:
            return c;
    }
}
```
