# 删除链表的倒数第n个节点

## 问题描述

给定一个链表，删除链表的倒数第 n 个节点，并且返回链表的头结点。

示例：

``` text
给定一个链表: 1->2->3->4->5, 和 n = 2.

当删除了倒数第二个节点后，链表变为 1->2->3->5.
```

**说明：**

给定的 n 保证是有效的。

**进阶：**

你能尝试使用一趟扫描实现吗？

## 解答

扫描两边的方法：第一遍确定链表的长度、第二遍找到要删除节点的前一个节点

``` java
public static ListNode solution(ListNode head, int n) {
    int size = 0;
    ListNode a = head;
    // 确定长度
    while (a != null) {
        a = a.next;
        size++;
    }
    a = head;
    // 遍历到倒数第n个节点的前一个节点
    for (int i = 0; i < size - n - 1; i++) {
        a = a.next;
    }
    // 边界的判断：删除的是第一个
    if (size == n) {
        return head.next;
    } else {
        a.next = a.next.next;
    }
    return head;
}
```

可以增加一个哑节点指向head，这样可以免除边界的判断

``` java
public static ListNode solution1(ListNode head, int n) {
    ListNode dummy = new ListNode(0);
    dummy.next = head;
    ListNode first = head;
    int length = 0;
    while (first != null) {
        length++;
        first = first.next;
    }
    length -= n;
    first = dummy;
    // 遍历到倒数第n个节点的前一个节点
    while (length != 0) {
        length--;
        first = first.next;
    }
    first.next = first.next.next;
    return dummy.next;
}
```

只遍历一遍的方法：使用两个指针，两个都指向head，第一个指针先移动`n+1步`，这样两个指针之间间隔了n个节点，然后两个一起移动，当第一个到达最后null时，第二节点刚好指向倒数第n个节点前的一个节点，将其next指向下下个节点即可。

``` java
public static ListNode solution2(ListNode head, int n) {
    ListNode dummy = new ListNode(0);
    dummy.next = head;
    ListNode first = dummy;
    ListNode second = dummy;
    // 第一个指针移动n+1
    for (int i = 0; i <= n + 1; i++) {
        first = first.next;
    }
    // 两个指针一起移动，直到第一个到达null
    while (first != null) {
        first = first.next;
        second = second.next;
    }
    second.next = second.next.next;
    return dummy.next;
}
```
