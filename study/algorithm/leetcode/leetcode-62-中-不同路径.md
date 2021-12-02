### 题目

![](http://img.yelizi.top/b41d0a95-f5ad-4133-99bc-444f90400dc3.jpg$xyz)


### 题解

```Java
// 标准的动态规划
// 只能往右和往下，故第一行第一列的每格到的方案都只有1
// 剩下的，只需一步步的取格上合格左的方案和，到最后一格自然汇总所有方案合
public int uniquePaths(int m, int n) {
    int[][] dp = new int[n][m];

    // 第一行全部置为1
    for (int j = 0; j < m; j++) {
        dp[0][j] = 1;
    }

    // 第一列全部置为1
    for (int i = 0; i < n; i++) {
        dp[i][0] = 1;
    }

    // 开始遍历
    for (int i = 1; i < n; i++) {
        for (int j = 1; j < m; j++) {
            dp[i][j] = dp[i - 1][j] + dp[i][j - 1];
        }
    }

    return dp[n - 1][m - 1];
}
```