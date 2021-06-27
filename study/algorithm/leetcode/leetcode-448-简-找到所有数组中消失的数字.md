### 题目

给定一个范围在 1 ≤ a[i] ≤ n ( n = 数组大小 ) 的 整型数组，数组中的元素一些出现了两次，另一些只出现一次。

找到所有在 [1, n] 范围之间没有出现在数组中的数字。

您能在不使用额外空间且时间复杂度为O(n)的情况下完成这个任务吗? 你可以假定返回的数组不算在额外空间内。

示例:

```
输入:
[4,3,2,7,8,2,3,1]

输出:
[5,6]
```

### 解法

**使用额外空间的O(n)**

```Java
public List<Integer> findDisappearedNumbers(int[] nums) {
    int len = nums.length;
    int[] data = new int[len];
    for (int num : nums) {
        data[num - 1] = num;
    }
    List<Integer> list = new ArrayList<>();
    for (int i = 0; i < len; i++) {
        if (data[i] == 0) {
            list.add(i + 1);
        }
    }

    return list;
}
```


**不使用额外空间的O(n)**

时间效率虽也为O(n)，仍旧比借助额外空间的要多耗费一点

```Java
public List<Integer> findDisappearedNumbers(int[] nums) {
    int len = nums.length;

    // 原地置换
    int index = 0;
    while (index < len) {
    int num = nums[index];
    // 数字在其位or为0 next
    if (num == index + 1 || num == 0) {
    index++;
    }
    // 数字不在其位 compare and swap
    else {
    int tmp = nums[num - 1];
    // 其位数字等于其 置0 and next
    if (tmp == num) {
    nums[index++] = 0;
    }
    // 其位数字不等于其 swap and repeat
    else {
    nums[num - 1] = num;
    nums[index] = tmp;
    }
    }
    }

    // 遍历0位单元
    List<Integer> list = new ArrayList<>();
    for (int i = 0; i < len; i++) {
    if (nums[i] == 0) {
    list.add(i + 1);
    }
    }

    return list;
    }
```