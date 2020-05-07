# k个一组翻转链表

## 问题描述

给你一个链表，每 k 个节点一组进行翻转，请你返回翻转后的链表。

k 是一个正整数，它的值小于或等于链表的长度。

如果节点总数不是 k 的整数倍，那么请将最后剩余的节点保持原有顺序。

示例：

``` text
给你这个链表：1->2->3->4->5

当 k = 2 时，应当返回: 2->1->4->3->5

当 k = 3 时，应当返回: 3->2->1->4->5
```

说明：

* 你的算法只能使用常数的额外空间。
* 你不能只是单纯的改变节点内部的值，而是需要实际进行节点交换。

## 解答

将每k个节点当成一个链表进行翻转，最后不足k个时直接返回

``` java
public static ListNode solution(ListNode head, int k) {
    if (head == null || k == 1) {
        return head;
    }
    ListNode dummy = new ListNode(-1);
    dummy.next = head;
    ListNode pre = dummy;
    ListNode tail = dummy;
    while (true) {
        int count = 0;
        // 找到每次翻转的链表的tail
        while (tail != null && count != k) {
            count++;
            tail = tail.next;
        }
        // tail为null说明已不足k个，结束循环
        if (tail == null) break;
        // 翻转k个节点组成的链表
        // 使用尾插法，遍历将每个节点插入到tail后面
        ListNode head1 = pre.next;
        while (pre.next != tail) {
            ListNode cur = pre.next;
            pre.next = cur.next;
            cur.next = tail.next;
            tail.next = cur;
        }
        pre = head1;
        tail = head1;
    }
    return dummy.next;
}
```
