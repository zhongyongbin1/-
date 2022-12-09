### 题目： 
链接：https://leetcode.cn/problems/linked-list-cycle-ii/

### 思路
根据快慢指针的思路，快指针走两步，慢指针走一步，如果是遇到了快指针等于慢指针的情况，那么证明是有环的，需要跳出，如果慢指针或者快指针某一个的下一个元素为None，那么证明他是没有环的

### 代码

```python
# Definition for singly-linked list.
# class ListNode:
#     def __init__(self, x):
#         self.val = x
#         self.next = None

class Solution:
    def detectCycle(self, head: Optional[ListNode]) -> Optional[ListNode]:





```