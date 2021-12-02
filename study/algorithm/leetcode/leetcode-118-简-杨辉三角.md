### 题目

![](http://img.yelizi.top/94bdc951-e2f0-46fe-b266-b4528cf53daf.png$xyz)

    
### 解法


``` java
    public List<List<Integer>> generate(int numRows) {
        if (numRows < 1) {
            return Collections.emptyList();
        }

        List<List<Integer>> data = new ArrayList<>();
        data.add(Collections.singletonList(1));

        for (int i = 1; i < numRows; i++) {
            data.add(generate(data.get(i - 1), i + 1));
        }

        return data;
    }

    private List<Integer> generate(List<Integer> last, int level) {
        List<Integer> data = new ArrayList<>();
        data.add(1);
        for (int i = 1; i < level - 1; i++) {
            data.add(last.get(i - 1) + last.get(i));
        }
        data.add(1);

        return data;
    }
```
