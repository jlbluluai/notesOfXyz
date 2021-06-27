### 题目

给定一个二进制数组， 计算其中最大连续1的个数。

示例 1:

```
输入: [1,1,0,1,1,1]
输出: 3
解释: 开头的两位和最后的三位都是连续1，所以最大连续1的个数是 3.
```


### 解法

```Java
public int findMaxConsecutiveOnes(int[] nums) {
    int historyMax = 0;
    int max = 0;
    for (int num : nums) {
        if (num == 1) {
            max++;
        } else {
            historyMax = Math.max(historyMax, max);
            max = 0;
        }
    }

    return Math.max(historyMax, max);
}
```
