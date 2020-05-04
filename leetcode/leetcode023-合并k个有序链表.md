# 合并k个有序链表

## 问题描述

合并 k 个排序链表，返回合并后的排序链表。请分析和描述算法的复杂度。

示例:

``` text
输入:
[
  1->4->5,
  1->3->4,
  2->6
]
输出: 1->1->2->3->4->4->5->6
```

## 解答

方案1：根据合并两个有序链表的方案，可以遍历k个有序链表，将其依次合并起来  
思路很清晰，但执行起来很慢

``` java
public static ListNode solution(ListNode[] lists) {
    ListNode ans = null;
    for (int i = 0; i < lists.length; i++) {
        ans = MergeTwoSortedList(ans, lists[i]);
    }
    return ans;
}

// 合并两个有序链表
public static ListNode MergeTwoSortedList(ListNode l1, ListNode l2) {
    ListNode dummy = new ListNode(-1);
    ListNode tail = dummy;
    while (l1 != null && l2 != null) {
        if (l1.val > l2.val) {
            tail.next = l2;
            l2 = l2.next;
        } else {
            tail.next = l1;
            l1 = l1.next;
        }
        tail = tail.next;
    }
    tail.next = l1 == null ? l2 : l1;
    return dummy.next;
}
```

方案2：分治合并，将链表两两合并直到只剩一个链表

``` java
public static ListNode solution1(ListNode[] lists) {
    if (lists == null || lists.length == 0) {
        return null;
    }
    int len = lists.length;
    int left = 0;
    int right = len - 1;
    // 总的循环次数为right从len-1到1
    for (int i = 0; i < len - 1; i++) {
        lists[left] = MergeTwoSortedList(lists[left], lists[right]);
        left++;
        right--;
        if (left >= right) {
            left = 0;
        }
    }
    return lists[0];
}
```
