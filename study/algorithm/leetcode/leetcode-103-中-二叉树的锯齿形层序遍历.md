### 题目

给定一个二叉树，返回其节点值的锯齿形层序遍历。（即先从左往右，再从右往左进行下一层遍历，以此类推，层与层之间交替进行）。

例如：
给定二叉树 [3,9,20,null,null,15,7],

```
    3
   / \
  9  20
    /  \
   15   7
```

返回锯齿形层序遍历如下：

```
[
  [3],
  [20,9],
  [15,7]
]
```

### 题解

```Java
public List<List<Integer>> zigzagLevelOrder(TreeNode root) {
    List<List<Integer>> data = new ArrayList<>();
    dps(root, 1, data);
    return data;
}

private void dps(TreeNode node, int level, List<List<Integer>> data) {
    if (node == null) {
        return;
    }

    // 未有记载的层数 先new
    if (data.size() < level) {
        data.add(new ArrayList<>());
    }

    // 根据层数奇偶决定正插倒插
    if (level % 2 == 1) {
        data.get(level - 1).add(node.val);
    } else {
        data.get(level - 1).add(0, node.val);
    }

    // 前序遍历
    dps(node.left, level + 1, data);
    dps(node.right, level + 1, data);
}
```