### 题目

给定一个由若干 0 和 1 组成的数组 A，我们最多可以将 K 个值从 0 变成 1 。

返回仅包含 1 的最长（连续）子数组的长度。

 

示例 1：
```
输入：A = [1,1,1,0,0,0,1,1,1,1,0], K = 2
输出：6
解释： 
[1,1,1,0,0,1,1,1,1,1,1]
粗体数字从 0 翻转到 1，最长的子数组长度为 6。
```
示例 2：
```
输入：A = [0,0,1,1,0,0,1,1,1,0,1,1,0,0,0,1,1,1,1], K = 3
输出：10
解释：
[0,0,1,1,1,1,1,1,1,1,1,1,0,0,0,1,1,1,1]
粗体数字从 0 翻转到 1，最长的子数组长度为 10。
```

    
### 解法

**暴力贪心**

```Java
public int longestOnes(int[] A, int K) {
    int historyMax = 0;
    int l = A.length;
    int prev = 0;
    for (int i = 0; i < l; i++) {
        // 前辈和自己都是1 直接过
        if (prev == 1) {
            continue;
        }
        prev = A[i];

        // 历史最大都超过剩下总长度了 直接结束
        if (historyMax > l - i) {
            break;
        }
        int k = K;
        int max = 0;
        for (int j = i; j < l; j++) {
            if (A[j] == 0) {
                if (k-- > 0) {
                    max += 1;
                } else {
                    break;
                }
            } else {
                max += 1;
            }
        }
        historyMax = Math.max(historyMax, max);
    }

    return historyMax;
}
```