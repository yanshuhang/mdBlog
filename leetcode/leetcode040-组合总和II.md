# 组合总和II

## 问题描述

给定一个数组 candidates 和一个目标数 target ，找出 candidates 中所有可以使数字和为 target 的组合。

candidates 中的每个数字在每个组合中只能使用一次。

## 解答

与组合总和相比只多了一个判断进行去重复的操作

``` java
public static List<List<Integer>> solution(int[] candidates, int target) {
    if (candidates == null || candidates.length == 0) {
        return Collections.emptyList();
    }
    List<List<Integer>> ans = new ArrayList<>();
    List<Integer> cur = new ArrayList<>();
    int start = 0;
    dfs(ans, cur, candidates, start, target);
    return ans;
}

private static void dfs(List<List<Integer>> ans, List<Integer> cur, int[] candidates, int start, int target) {
    if (target == 0) {
        ans.add(new ArrayList<>(cur));
        return;
    }
    for (int i = start; i < candidates.length; i++) {
        if (candidates[i] > target) {
            break;
        }
        // i>start说明:i-1在回溯中剪掉了,也就是说以i-1为根节点的所有的分枝都已经判断完毕了,现在将i-1剪掉,换成i为根节点来判断所有的分支
        // 如果i处的值和i-1处的值相同,那么i的分支都已经在i-1分支里了,不需要判断了
        // 这样就把重复的情况去除了
        if (i > start && candidates[i] == candidates[i - 1]) {
            continue;
        }
        cur.add(candidates[i]);
        dfs(ans, cur, candidates, i + 1, target - candidates[i]);
        cur.remove(cur.size() - 1);
    }
}
```
