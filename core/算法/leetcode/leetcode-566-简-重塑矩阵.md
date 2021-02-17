### 题目

在MATLAB中，有一个非常有用的函数 reshape，它可以将一个矩阵重塑为另一个大小不同的新矩阵，但保留其原始数据。

给出一个由二维数组表示的矩阵，以及两个正整数r和c，分别表示想要的重构的矩阵的行数和列数。

重构后的矩阵需要将原始矩阵的所有元素以相同的行遍历顺序填充。

如果具有给定参数的reshape操作是可行且合理的，则输出新的重塑矩阵；否则，输出原始矩阵。

示例 1:

```
输入:
nums =
[[1,2],
[3,4]]
r = 1, c = 4
输出:
[[1,2,3,4]]
解释:
行遍历nums的结果是 [1,2,3,4]。新的矩阵是 1 * 4 矩阵, 用之前的元素值一行一行填充新矩阵。
```

示例 2:

```
输入:
nums =
[[1,2],
[3,4]]
r = 2, c = 4
输出:
[[1,2],
[3,4]]
解释:
没有办法将 2 * 2 矩阵转化为 2 * 4 矩阵。 所以输出原矩阵。
```

### 解法

``` java
public int[][] matrixReshape(int[][] nums, int r, int c) {
    int or = nums.length;
    int oc = nums[0].length;
    if (or * oc != r * c) {
        return nums;
    }

    int[][] data = new int[r][c];
    // 凑行
    if (r % or == 0) {
        // 原本一行分化的行数
        int x = r / or;
        for (int i = 0; i < or; i++) {
            int start = i * x;
            for (int j = 0; j < oc; j++) {
                int a = j / c;
                data[start + a][j - a * c] = nums[i][j];
            }
        }
    }
    // 凑列
    else if (c % oc == 0) {
        // 原本几行凑一行
        int x = c / oc;
        for (int i = 0; i < or; i++) {
            int start = (i % x) * oc;
            for (int j = 0; j < oc; j++) {
                data[i / x][start + j] = nums[i][j];
            }
        }
    } else {
        Queue<Integer> queue = new LinkedList<>();
        for (int i = 0; i < or; i++) {
            for (int j = 0; j < oc; j++) {
                queue.add(nums[i][j]);
            }
        }
        for (int i = 0; i < r; i++) {
            for (int j = 0; j < c; j++) {
                data[i][j] = queue.poll();
            }
        }
    }

    return data;
}
```
