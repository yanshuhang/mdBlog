# 两数相加

## 问题描述

给出两个 非空 的链表用来表示两个非负的整数。其中，它们各自的位数是按照 **逆序** 的方式存储的，并且它们的每个节点只能存储 一位 数字。  
如果，我们将这两个数相加起来，则会返回一个新的链表来表示它们的和。  
您可以假设除了数字 0 之外，这两个数都不会以 0 开头。

示例：

``` text
输入：(2 -> 4 -> 3) + (5 -> 6 -> 4)
输出：7 -> 0 -> 8
原因：342 + 465 = 807
```

## 解答

``` java
public class AddTwoNumbers {
    public static ListNode solution(ListNode l1, ListNode l2) {
        if (l1 == null) {
            return l2;
        }
        if (l2 == null) {
            return l1;
        }
        // 用于记录返回的节点
        ListNode result = new ListNode(0);
        // 用于while循环中记录下一个节点
        ListNode head = result;
        int n = 0;

        // 当l1、l2都遍历完，且进位的n也为0时结束循环
        while (l1 != null || l2 != null || n != 0) {
            if (l1 != null) {
                n += l1.value;
                l1 = l1.next;
            }
            if (l2 != null) {
                n += l2.value;
                l2 = l2.next;
            }

            // 当前位的值放到listNode首位
            ListNode nextNode = new ListNode(n % 10);
            head.next = nextNode;
            head = nextNode;

            // 进位的值用于下次循环
            n = n / 10;
        }

        return result.next;
    }

    static class ListNode {
        int value;
        ListNode next;

        public ListNode(int value) {
            this.value = value;
        }
    }
}

```
