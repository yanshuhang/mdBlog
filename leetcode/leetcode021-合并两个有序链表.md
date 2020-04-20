# 合并两个有序链表

## 问题描述

将两个升序链表合并为一个新的升序链表并返回。新链表是通过拼接给定的两个链表的所有节点组成的。

示例：

``` text
输入：1->2->4, 1->3->4
输出：1->1->2->3->4->4
```

## 解答

递归法：每一层减去一个较小的节点，直到某个链表为null递归结束。

``` java
public static ListNode solution(ListNode l1, ListNode l2) {
    if (l1 == null) {
        return l2;
    } else if (l2 == null) {
        return l1;
    } else if (l1.val < l2.val) {
        l1.next = solution(l1.next, l2);
        return l1;
    } else {
        l2.next = solution(l1, l2.next);
        return l2;
    }
}
```

遍历法：两个指针，一个指向head前的节点不动，一个链接节点之后向后移动，最后返回preHead.next即可

``` java
public static ListNode solution1(ListNode l1, ListNode l2) {
    ListNode preHead = new ListNode(-1);
    ListNode pre = preHead;
    while (l1 != null && l2 != null) {
        if (l1.val <= l2.val) {
            pre.next = l1;
            l1 = l1.next;
        } else {
            pre.next = l2;
            l2 = l2.next;
        }
        pre = pre.next;
    }
    pre.next = l1 == null ? l2 : l1;
    return preHead.next;
}
```
