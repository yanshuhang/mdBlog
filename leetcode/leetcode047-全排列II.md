# 全排列

## 问题描述

给定一个可包含重复数字的序列 nums ，按任意顺序 返回所有不重复的全排列。

## 解答

回溯法:

* 由于nums有重复数字,且结果不能有重复的全排列,需要去重,先排序数组,这样相同的数字都相邻,在循环中可简单的去重复
* 回溯算法:参数-总的结果ans,单个结果cur,nums数组,记录当前索引的数字是否使用过的数字used,已使用数字的数量count
* count等于数组长度是,将当前结果cur加入到ans中
* 在for循环中去重:1.已使用过该索引 2.值跟前面的相等且前面的索引在本次结果中还未使用  
条件1很好理解-本次用过的索引不能再用  
条件2说明相同的情况已经在另外的结果中求解过了,再解一次是重复的

``` java
public List<List<Integer>> solution(int[] nums) {
    if (nums == null || nums.length == 0) {
        return Collections.emptyList();
    }
    Arrays.sort(nums);
    boolean[] used = new boolean[nums.length];
    List<List<Integer>> ans = new ArrayList<>();
    List<Integer> cur = new ArrayList<>();
    dfs(ans, cur, nums, used, 0);
    return ans;
}

public void dfs(List<List<Integer>> ans, List<Integer> cur, int[] nums, boolean[] used, int count) {
    if (count == nums.length) {
        ans.add(new ArrayList<>(cur));
        return;
    }
    for (int i = 0; i < nums.length; i++) {
        // 使用过该索引或者与前面的相等且前面的索引在本次结果中还未使用,说明是重复的
        if (used[i] || (i > 0 && nums[i] == nums[i - 1] && !used[i - 1])) {
            continue;
        }
        cur.add(nums[i]);
        used[i] = true;
        dfs(ans, cur, nums, used, count+1);
        used[i] = false;
        cur.remove(count);
    }
}
```
