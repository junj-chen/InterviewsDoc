高频Leetcode

1. 反转链表

   https://www.nowcoder.com/practice/75e878df47f24fdc9dc3e400ec6058ca?tpId=188&&tqId=38547&rp=1&ru=/activity/oj&qru=/ta/job-code-high-week/question-ranking

~~~java

    public ListNode ReverseList(ListNode head) {
        
        if(head == null || head.next == null) return head;
        
        ListNode node = ReverseList(head.next);  // 进行递归到最后一个节点
        
        // 此时的 head 为最后一个节点的上一个节点
        head.next.next = head;  //
        head.next = null;  // 防止出现环
        
        return node;
    }
~~~



2. 合并两个有序链表

~~~java
// https://www.nowcoder.com/practice/d8b6b4358f774294a89de2a6ac4d9337?tpId=188&&tqId=38642&rp=1&ru=/activity/oj&qru=/ta/job-code-high-week/question-ranking


    public ListNode Merge(ListNode list1,ListNode list2) {
        
        ListNode pre = new ListNode(-1);  // 定义虚拟头结点
        
        ListNode res = pre; // 定义移动的节点
        
        ListNode curA = list1;
        ListNode curB = list2;
        
        while(curA != null && curB != null){
            
            if(curA.val < curB.val){
                res.next = curA;
                res = curA;
                curA = curA.next;
            }else{
                res.next = curB;
                res = curB;
                curB = curB.next;
            }
        }
        
        // 最后防止链表没有遍历完成
        res.next = (curA == null ? curB : curA);
        
        return pre.next;
    }

// 解法二
    public ListNode Merge(ListNode list1,ListNode list2) {
        
        // 使用递归的方式实现
        if(list1 == null) return list2;
        if(list2 == null) return list1;
        
        if(list1.val < list2.val){
            list1.next = Merge(list1.next, list2);
            return list1;
        }else{
            list2.next = Merge(list1, list2.next);
            return list2;
        }
        
    }


~~~



3. #### [ 删除链表的倒数第 N 个结点](https://leetcode-cn.com/problems/remove-nth-node-from-end-of-list/)

~~~java



~~~



















































































































