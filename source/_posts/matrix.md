---
title: leetcode练习——01 matrix
date: 2018-08-11 18:52:59
tags: 算法 宽度优先搜索
---

https://leetcode-cn.com/problems/01-matrix/description/

01 矩阵

给定一个矩阵，其中的元素只有0和1，计算每个元素最近“0”的距离。

计算距离的时候，矩阵只有4个方向（上下左右）。

临近元素的距离是1。

<!-- more -->

**Example 1: **
Input:

```
0 0 0
0 1 0
0 0 0

```

Output:

```
0 0 0
0 1 0
0 0 0

```

**Example 2: **
Input:

```
0 0 0
0 1 0
1 1 1

```

Output:

```
0 0 0
0 1 0
1 2 1
```



在解题的时候一直被题目分类“深度优先搜索”影响，一直在尝试使用深度优先搜索的思想解题。能想到的方式是对给定元素{i, j}从上下左右四个方向出发找到最邻近的“0”，取最小的距离。但使用这种方式，每次计算都只为矩阵中的一个”1“，需要遍历整个矩阵的所有”1“才能得到最终答案。

说个题外话，在写代码的过程中犯了值得提醒自己的错误：`在深度优先搜索的时候忘了存储哪些已经搜索过，导致搜索会不停止，出现stackoverflow`。

尽管后来纠正了这个错误，也还是会TLE。因为这种方式的最坏情况时间复杂度是O((r * c)<sup>2</sup>)。



参考的答案解法是使用BFS + 队列的方式。

我们使用逆向思维从”0“出发。矩阵中“0”上下左右的“1”其最近"0"距离肯定是1。0附近“1”附近的“1”最邻近距离可能是2。之所以是可能，因为我们无法确定是不是存在更短的距离，这个需要比较。

```
x ? ?
1 y ?
0 1 z
```

仅看上图，我们能确定最左下角”0“附近的”1“最邻近距离是1，但是这些"1"邻近的元素x、y、z不一定最近"0"距离是2，因为x、y、z旁边可能有"0"。所以我们无法在该情况下直接算x，y，z的最近“0”距离，必须要先算好其他“0"，才能算x、y、z。

我们需要按顺序来计算元素的最近”0"距离。建立一个队列，将当前已知最近”0“距离按照距离大小放入队列中，距离小的先入，距离大的后入。这样我们能保证每次在当前算上下左右四个方向元素的最近“0”距离时，是最小的（注意是当前）。



算法基本过程：

* 将结果矩阵dist[i][j]全部元素初始化为INT_MAX。
* 遍历一次矩阵，将所有的”0“对应的坐标{i, j}置入队列。因为这些“0”的最近"0"距离就是0，是目前已知的最小距离元素。同时更新dist矩阵中更新这些”0“对应的结果为0。
* 每次从队列中取出一个元素，比较上下左右四个方向的元素是否通过该元素找到最近”0“的距离会更小。是，则更新对应方向的邻近元素最近”0“距离为该元素最近“0”距离 + 1；不是，则忽略。
* 遍历直到队列中没有元素，即找完了

该解法的关键是从“0”出发，而不是从“1”出发。同时将目前已知最小的最近“0”距离矩阵元素放到队列中，建立了顺序。



参考代码：

```java
private class Pair {
  int r;
  int c;

  public Pair(int r, int c) {
    this.r = r;
    this.c = c;
  }

  public int getR() {
    return r;
  }

  public int getC() {
    return c;
  }
}

public int[][] updateMatrix(int[][] m) {
  Queue<Pair> queue = new LinkedList<>();
  int[][] r = new int[m.length][m[0].length];
  for (int i = 0; i < m.length; i++) {
    for (int j = 0; j < m[i].length; j++) {
      if (m[i][j] == 0) {
        r[i][j] = 0;
        queue.add(new Pair(i, j));
      } else {
        r[i][j] = Integer.MAX_VALUE;
      }
    }
  }

  int[][] dir = {{-1, 0}, {0, -1}, {1, 0}, {0, 1}};
  while (!queue.isEmpty()) {
    Pair pair = queue.poll();
    System.out.println(pair.getR() + ", " + pair.getC());
    for (int i = 0; i < 4; i++) {
      int newR = pair.getR() + dir[i][0];
      int newC = pair.getC() + dir[i][1];
      if (newR >= 0 && newR < m.length && newC >= 0 && newC < m[0].length) {
        if (r[pair.getR()][pair.getC()] + 1 < r[newR][newC]) {
          r[newR][newC] = r[pair.getR()][pair.getC()] + 1;
          queue.add(new Pair(newR, newC));
        }
      }
    }
  }

  return r;
}
```

