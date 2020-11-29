# 组合总和

## 问题描述

给定一个无重复元素的数组 candidates 和一个目标数 target ，找出 candidates 中所有可以使数字和为 target 的组合。

candidates 中的数字可以无限制重复被选取。

## 解答

在排序数组中遍历,回溯剪枝:

* 如果匹配了,将当前的结果存到最终结果中
* 匹配过程中如果当前值比target大,中断循环,这个就是排序后可以省略很多不必要的遍历
* 将当前值添加到列表中,然后进入下一层的求解
* 从下一层返回后,不论是否匹配成功,都要把上一个值从列表中删除,也就是回溯剪枝

``` java
public static List<List<Integer>> solution(int[] candidates, int target) {
    if (candidates == null || candidates.length == 0) {
        return Collections.emptyList();
    }
    // 排序数组
    Arrays.sort(candidates);
    // ans存储最后结果,cur存储单枝的结果
    List<List<Integer>> ans = new ArrayList<>();
    List<Integer> cur = new ArrayList<>();
    dfs(ans, cur, candidates, 0, target);
    return ans;
}

private static void dfs(List<List<Integer>> ans, List<Integer> cur,int[] candidates, int start, int target) {
    // 匹配后将cur列表添加到ans中
    // 注意是从cur中复制元素到新列表中
    // 然后方法返回
    if (target == 0) {
        ans.add(new ArrayList<>(cur));
        return;
    }
    for (int i = start; i < candidates.length; i++) {
        // 如果当前比target大,数组已排序,后面的也不会合适,中断循环 方法返回
        if (candidates[i] > target) {
            break;
        }
        // 合适就加入列表中
        cur.add(candidates[i]);
        // 然后target减去当前值继续求解
        dfs(ans, cur, candidates, i, target - candidates[i]);
        // 递归方法返回后会执行这一步
        // 两个返回方式:
        // 1.找到结果return返回
        // 2.不匹配提前中断循环break返回
        // 两个返回都需要将上一个值删除,进行下一个值的匹配
        cur.remove(cur.size() - 1);
    }
}
```
