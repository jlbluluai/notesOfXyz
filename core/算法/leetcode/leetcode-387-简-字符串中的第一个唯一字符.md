### 题目

给定一个字符串，找到它的第一个不重复的字符，并返回它的索引。如果不存在，则返回 -1。

示例：

```
s = "leetcode"
返回 0

s = "loveleetcode"
返回 2
```

### 解法

``` java
// 速度不太可 尚需优化
public int firstUniqChar(String s) {
    Map<Character, Integer> map = new LinkedHashMap<>();
    char[] chars = s.toCharArray();

    int l = chars.length;
    for (int i = 0; i < l; i++) {
        int value = map.getOrDefault(chars[i], 0);
        map.put(chars[i], value + l + i);
    }

    int l2 = l << 1;
    int index = -1;
    for (Map.Entry<Character, Integer> entry : map.entrySet()) {
        if (entry.getValue() < l2) {
            index = entry.getValue() - l;
            break;
        }
    }
    return index;
}
```
