### 题目

有一个二维矩阵 A 其中每个元素的值为 0 或 1 。

移动是指选择任一行或列，并转换该行或列中的每一个值：将所有 0 都更改为 1，将所有 1 都更改为 0。

在做出任意次数的移动后，将该矩阵的每一行都按照二进制数来解释，矩阵的得分就是这些数字的总和。

返回尽可能高的分数。


示例：

```
输入：[[0,0,1,1],[1,0,1,0],[1,1,0,0]]
输出：39
解释：
转换为 [[1,1,1,1],[1,0,0,1],[1,1,1,1]]
0b1111 + 0b1001 + 0b1111 = 15 + 9 + 15 = 39
```

    
### 解法（贪心）

```Java
public int matrixScore(int[][] A) {
    if (A.length < 1) {
        return 0;
    }

    int row = A.length;
    int col = A[0].length;

    // 将行首位不是1的行置换，保证每行都是最大
    for (int i = 0; i < row; i++) {
        if (A[i][0] == 0) {
            for (int j = 0; j < col; j++) {
                A[i][j] = 1 - A[i][j];
            }
        }
    }

    // 列保证1数量多于0即可保证往大靠拢
    int halfRow = row >> 1;
    for (int j = 0; j < col; j++) {
        int count0 = 0;
        for (int[] ints : A) {
            if (ints[j] == 0) {
                count0++;
            }
        }
        if (count0 > halfRow) {
            for (int i = 0; i < row; i++) {
                A[i][j] = 1 - A[i][j];
            }
        }
    }

    // 依次取每行做和
    int sum = 0;
    for (int[] ints : A) {
        StringBuilder builder = new StringBuilder();
        for (int j = 0; j < col; j++) {
            builder.append(ints[j]);
        }
        sum += Integer.parseInt(builder.toString(), 2);
    }

    return sum;
}
```