---
title: leetcode练习——Remove Boxes
date: 2018-08-04 15:51:24
tags: leetcode 动态规划 搜索
---

# 题目

题目链接： https://leetcode-cn.com/problems/remove-boxes/description/

题目大意：

给定一个数组，你可以消除其中连续的数字，每次消除k个连续的数字（k>=1)，能得到k*k的积分，求该数组能达到的最大积分。

<!-- more -->

例如：

>Input:
>
>```
>[1, 3, 2, 2, 2, 3, 4, 3, 1]
>
>```
>
>Output:
>
>```
>23
>
>```
>
>Explanation:
>
>```
>[1, 3, 2, 2, 2, 3, 4, 3, 1] 
>----> [1, 3, 3, 4, 3, 1] (3*3=9 points) 
>----> [1, 3, 3, 3, 1] (1*1=1 points) 
>----> [1, 1] (3*3=9 points) 
>----> [] (2*2=4 points)
>```



# 失败的尝试

这道题在不使用动态规划和搜索之前，我的思路是构造递归使用暴力解法。我们都知道为了更高的积分，肯定是要不断消除那些单个的数字，直至出现比较多的连续数字，然后一下子消除。例如上文的例子中，数组中的“3”被”4“隔断，所以要消除”4“。

所以一开始我的想法是找到数组中出现次数最多的数字"x"，消除每个"x"之间的子序列。

> […x,…, x,…,x….]

最大的积分应该是：

f(0, x1) + f(x1, x2) + f(x3, x4) + … + N<sub>x</sub> * N<sub>x</sub>

N<sub>x</sub>代表"x"出现的次数

f(i, j)代表数组中第i个元素和第j个元素之间的最大积分

x1, x2, …, xn代表“x"在数组中的元素位置

但事实是，找出最多数字"x"并消除得到的积分不一定是最大的，例如：

> [3, 1, 2, 3, 2, 1, 3]

找出数组中出现次数最多的数字并没有用。但起码我们知道，一定要找出数组中出现次数多于1的数字，不然积分等于数组的长度。

解题的思路变为找到数组中所有出现次数多于1的数字，然后按照上文提及的消除子序列方法，计算假如要消除该数字，达到的最大的积分。

即遍历数组中出现次数大于1的数字，找到这些数字在数组中的位置，递归地计算：

f(0, x1) + f(x1, x2) + f(x3, x4) + … + N<sub>x</sub> * N<sub>x</sub>

这种方法是可行的，但是时间太长，超过了题目的时间限制



# 参考的解法：动态规划+搜索

> s…x….{x,x,x,…,x}     // 最右边都是连续的x

当数组最右出现连续的数字时，我们肯定是要去消除，但是要算出数组最大积分，会有两种场景：

1. 仅消除最右边连续的"x"，不理会左边的元素
2. 消除最右边连续的"x"以及数组左边中的若干个"x"



最主要考虑的是第2种场景：

考虑这样的数组：

> S…x1…x2…x3..x4x5x6

右边的连续元素肯定要和左边数组中某几个元素消除，但是消除哪个的积分最大并不清楚。我们从右往左来看：

右边连续元素x4, x5,x6和x3及之前的数组消除，会得到[s...x3, x4,x5,x6] +  {x3 ~ x4之间数组}最大积分。由于x1和x2元素排列在x3元素之前，计算[s...x3, x4,x5,x6] 最大积分的时候会考虑x1和x2对整个积分的影响。

右边连续元素x4,x5, x6和x2及之前的数组消除，会得到[s...x2, x4,x5,x6] + {x2 ~ x4之间数组}最大积分，忽略了x3。

右边连续元素x4,x5, x6和x1及之前的数组消除，会得到[s...x1, x4,x5,x6]+ {x1 ~ x4之间数组}最大积分，忽略了x2和x3。

这是在消除的时候能覆盖的所有情况： x1, x2, x3,…xn由右往左地覆盖所有可能，忽略xn, xn-1, xn-2直到x1。



为了减少时间，存储计算过的结果[s...x1,x2,x3,...xn]。这样下次遇到同样的区间，并且最有元素个数都相同的时候，就能直接获取结果。存储结构用d{l,r,k}表示，l表示数组中的最左元素序号，r代表数组中最右元素的序号，k代表boxes[r]之后有k个连续的数字，并且该数字等于boxes[r]。如d{0, 2, 1代表如下}:

> z, y, x, x



# 代码实现

```java
class Solution {
    public int removeBoxes(int[] boxes) {
        int[][][] results = new int[100][100][100];
        return dfs(boxes, results, 0, boxes.length - 1, 0);
    }
    
    private int dfs(int[] boxes, int[][][] results, int left, int right, int duplicateNum) {
        if (left > right) {
            return 0;
        }
        
        if (results[left][right][duplicateNum] > 0) {
            return results[left][right][duplicateNum];
        }
        
        // 第一种情况，只考虑最右边的所有重复数字
        while(left < right && boxes[right] == boxes[right - 1]) {
            right--;
            duplicateNum++;
        }
        results[left][right][duplicateNum] = (duplicateNum + 1) * (duplicateNum + 1) + dfs(boxes, results, left, right-1, 0);
        
        // 第二种情况，考虑左边的数字
        for (int i = left; i < right; i++) {
            if (boxes[i] == boxes[right]) {
                results[left][right][duplicateNum] = Math.max(results[left][right][duplicateNum], dfs(boxes, results, left, i, duplicateNum + 1) + dfs(boxes, results, i + 1, right - 1, 0));
            }
        }
        return results[left][right][duplicateNum];
    }
}
```

