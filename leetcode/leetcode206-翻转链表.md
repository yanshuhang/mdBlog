# 反转链表

## 问题描述

反转一个单链表。

示例:

``` text
输入: 1->2->3->4->5->NULL
输出: 5->4->3->2->1->NULL
进阶:
你可以迭代或递归地反转链表。你能否用两种方法解决这道题？
```

## 解答

迭代法：双指针，pre指向前一个节点，cur指向当前节点，每次遍历将cur.next指向pre，然后pre和cur一起向后移动

``` java
private ListNode solution1(ListNode head) {
    if (head == null || head.next == null) return head;
    ListNode pre = null;
    ListNode cur = head;
    while (cur != null) {
        ListNode next = cur.next;
        cur.next = pre;
        pre = cur;
        cur = next;
    }
    return pre;
}
```

递归法：方法反转链表返回head，方法递归相当于遍历到尾部结束，然后向前反转next的指向

``` java
private ListNode solution2(ListNode head) {
    if (head == null || head.next == null) {
        return head;
    }
    ListNode p = solution2(head.next);
    head.next.next = head;
    head.next = null;
    return p;
}
```
