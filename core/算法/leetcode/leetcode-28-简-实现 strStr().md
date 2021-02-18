### 题目

实现 strStr() 函数。

给定一个 haystack 字符串和一个 needle 字符串，在 haystack 字符串中找出 needle 字符串出现的第一个位置 (从0开始)。如果不存在，则返回  -1。

示例 1:
```
输入: haystack = "hello", needle = "ll"
输出: 2
```
示例 2:
```
输入: haystack = "aaaaa", needle = "bba"
输出: -1
```

### 解法


``` java
public int strStr(String haystack, String needle) {
    if ("".equals(needle)) {
        return 0;
    }

    int hl = haystack.length();
    int nl = needle.length();
    char[] hc = haystack.toCharArray();
    char[] nc = needle.toCharArray();
    for (int i = 0; i < hl - nl + 1; i++) {
        boolean flag = false;
        for (int j = 0; j < nl; j++) {
            if (hc[i + j] != nc[j]) {
                flag = true;
                break;
            }
        }
        if (!flag) {
            return i;
        }
    }

    return -1;
}
```
