# 旋转链表

## 问题描述

给定一个链表，旋转链表，将链表每个节点向右移动 k 个位置，其中 k 是非负数。

示例 1:

``` text
输入: 1->2->3->4->5->NULL, k = 2
输出: 4->5->1->2->3->NULL
解释:
向右旋转 1 步: 5->1->2->3->4->NULL
向右旋转 2 步: 4->5->1->2->3->NULL
```

示例 2:

``` text
输入: 0->1->2->NULL, k = 4
输出: 2->0->1->NULL
解释:
向右旋转 1 步: 2->0->1->NULL
向右旋转 2 步: 1->2->0->NULL
向右旋转 3 步: 0->1->2->NULL
向右旋转 4 步: 2->0->1->NULL
```

## 解答

1. 将链表首尾相连，形成一个环形链表
2. 顺便求得链表的大小
3. 根据大小和移动的位置k，找到新的尾部、头部节点
4. 将新的尾部、头部节点之间断开即可

``` java
public ListNode rotateRight(ListNode head, int k) {
    if (head == null || head.next == null) return head;
    ListNode old_tail = head;
    int size = 1;
    while (old_tail.next != null) {
        old_tail = old_tail.next;
        size++;
    }
    old_tail.next = head;
    ListNode new_tail = head;
    for (int i = 0; i < size - k%size - 1; i++) {
        new_tail = new_tail.next;
    }
    ListNode new_head = new_tail.next;
    new_tail.next = null;
    return new_head;
}
```
