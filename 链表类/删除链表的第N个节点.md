### 题目： 
链接：https://leetcode.cn/problems/remove-nth-node-from-end-of-list/

### 思路
根据快慢指针的思路，让快指针先走k步骤，如果在k步骤及以内走完链表，那么是空的，等快指针到k，然后将继续走慢指针（从头开始）
### 注意点
如果涉及删除节点问题，一般设计一个头节点。这样可以防止删除到最后的时候空的情况

### 代码

```python
# Definition for singly-linked list.
# class ListNode:
#     def __init__(self, x):
#         self.val = x
#         self.next = None

class Solution:
    def removeNthFromEnd(self, head: Optional[ListNode], n: int) -> Optional[ListNode]:
        fast, slow = head, ListNode(0)
        slow.next = head
        if not head:
            return head
        for i in range(k):
            if not fast:
                break
            fast = fast.next
        while fast:
            slow = slow.next
            fast = fast.next
        slow.next = slow.next.next
        return head

```

### 关联题目
https://leetcode.cn/problems/shan-chu-lian-biao-de-jie-dian-lcof/description/