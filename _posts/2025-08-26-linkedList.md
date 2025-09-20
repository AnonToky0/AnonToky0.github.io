---
title: "链表"
date: 2025-08-26 17:28:00 +0800
categories: [数据结构与算法]
tags: [链表，数据结构]
---

## Definition

``` cpp
struct ListNode{
    int val;
    ListdNode * next;
    ListNode() : val(0), next(nullptr) {}
    ListNode(int x) : val(x), next(nullptr){}
    ListNode(int x, ListNode* next): val(x), next(next) {} 
};
```

- 构建
``` cpp 
ListNode *createList(const initializer_list<int> &vals){
    ListNode dummy = ListNode();
    ListNode *tail = &dummy;
    for(int val: vals){
        tail ->next = new ListNode(val);
        tail = tail ->next;
    }
    return dummy.next;
}

```

打印
``` cpp
void printList(ListNode *head){
    while(head){ 
        cout << head->val;
        if (head ->next)
        {
            cout << " -> ";
            head = head->next;
        }
    }
    cout << endl;
}
```


- [160.相交链表](https://leetcode.cn/problems/intersection-of-two-linked-lists/description/)
