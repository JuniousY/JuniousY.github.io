---
title: LeetCode 23. Merge k Sorted Lists
date: 2019-05-17 16:53:24
categories: 刷题
tags: 算法
---

Merge *k* sorted linked lists and return it as one sorted list. Analyze and describe its complexity.

**Example:**

```
Input:
[
  1->4->5,
  1->3->4,
  2->6
]
Output: 1->1->2->3->4->4->5->6
```

<!-- more -->



```java
/*
 * @lc app=leetcode id=23 lang=java
 *
 * [23] Merge k Sorted Lists
 */
/**
 * Definition for singly-linked list.
 * public class ListNode {
 *     int val;
 *     ListNode next;
 *     ListNode(int x) { val = x; }
 * }
 */
class Solution {
    public ListNode mergeKLists(ListNode[] lists) {
        if (lists == null || lists.length == 0) return null;
        PriorityQueue<ListNode> pQueue = new PriorityQueue<>((a, b) -> a.val - b.val);
        ListNode dummy = new ListNode(0);
        ListNode cur = dummy;
        for (ListNode node:lists) {
            if (node == null) continue;
            pQueue.offer(node);
        }
        while(!pQueue.isEmpty()) {
            cur.next = pQueue.poll();
            cur = cur.next;
            if (cur.next != null) pQueue.offer(cur.next);
        }
        return dummy.next;
    }
}
```



Time complexity: O(nlogk)

Space complexity: O(k)