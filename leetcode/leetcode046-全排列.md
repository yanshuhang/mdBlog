# 全排列

## 问题描述

给定一个`没有重复`数字的序列，返回其所有可能的全排列。

## 解答

回溯算法,主要在于一个数字只能使用一次,需要一个`boolean`数组记录数字是否使用过,具体见注释

``` java
public List<List<Integer>> solution(int[] nums) {
    if (nums == null || nums.length == 0) {
        return Collections.emptyList();
    }
    // ans存储全部结果,cur存储一个结果
    List<List<Integer>> ans = new ArrayList<>();
    List<Integer> cur = new ArrayList<>();
    // used记录nums的索引是否已经使用过
    boolean[] used = new boolean[nums.length];
    // 回溯算法
    dfs(ans, cur, nums, used, 0);
    return ans;
}

public void dfs(List<List<Integer>> ans, List<Integer> cur, int[] nums, boolean[] used, int count) {
    // 放完了之后,单次的结果加入到全部结果中,方法返回
    if (count == nums.length) {
        ans.add(new ArrayList<>(cur));
        return;
    }
    for (int i = 0; i < nums.length; i++) {
        // 如果索引i处的值已经用过,跳过,继续for循环
        if (used[i]) {
            continue;
        }
        // 添加可以用的值,并记录为已用过
        cur.add(nums[i]);
        used[i] = true;
        dfs(ans, cur, nums, used, count + 1);
        // 递归方法返回后,该数字已记录过,剪掉,used改为false,继续for循环
        used[i] = false;
        cur.remove(count);
    }
}
```
