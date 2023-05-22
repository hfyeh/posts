---
layout: post
date: 2022-10-28
title: "Linux 核心設計 L01 quiz1 (2020q1)"
tags: [blog, linked-list, draft]
draft: true
---

This is the homework of [jserv](https://github.com/jserv)'s [Linux Kernel Internals](https://wiki.csie.ncku.edu.tw/linux/schedule).

Reference: [Linked list quiz 2](https://hackmd.io/@sysprog/linux2020-quiz1)

<!--more-->

考慮一個單向 linked list:

```c
typedef struct __list {
    int data;
    struct __list *next;
} list;
```

在不存在環狀結構的狀況下，以下函式能夠對 linked list 元素從小到大排序:

```c
list *sort(list *start) {
    if (!start || !start->next)
        return start;
    list *left = start;
    list *right = left->next;
    LL0;

    left = sort(left);
    right = sort(right);

    for (list *merge = NULL; left || right; ) {
        if (!right || (left && left->data < right->data)) {
            if (!merge) {
                LL1;
            } else {
                LL2;
                merge = merge->next;
            }
            LL3;
        } else {
            if (!merge) {
                LL4;
            } else {
                LL5;
                merge = merge->next;
            }
            LL6;
        }
    }
    return start;
}
```

請補完程式碼。

#### 1. LL0=?
-   `(a)` `left->next = NULL`
-   `(b)` `right->next = NULL`
-   `(c)` `left = left->next`
-   `(d)` `left = right->next`

Since it's singly linked list, and it's separated into two lists, the connection between `left` and `right` should be cut down.

Answer: (a)

#### 2. LL1=?
-   `(a)` `start = left`
-   `(b)` `start = right`
-   `(c)` `merge = left`
-   `(d)` `merge = right`
-   `(e)` `start = merge = left`
-   `(f)` `start = merge = right`

In the first iteration `merge` is null. After comparing `left` and `right`, `merge` should be assigned to the lesser one. So (c), (e) is applied. (Since `start` is `left` in the beginning)

#### 3. LL2=?
-   `(a)` `merge->next = left`
-   `(b)` `merge->next = right`

Since `left` is less than `right`, `merge->next` should be overwritten to `left`.  Answer is (a).

#### 4. LL3=?
-   `(a)` `left = left->next`
-   `(b)` `right = right->next`
-   `(c)` `left = right->next`
-   `(d)` `right = left->next`

Now the first node is merged into `merge`, so we remove it from `left` by using (a).

#### 5. LL4=?
-   `(a)` `start = left`
-   `(b)` `start = right`
-   `(c)` `merge = left`
-   `(d)` `merge = right`
-   `(e)` `start = merge = left`
-   `(f)` `start = merge = right`

The logic is just like question 2, so the answer is (f).

#### 6. LL5=?
-   `(a)` `merge->next = left`
-   `(b)` `merge->next = right`

The logic is just like question 3, so the answer is (b).

#### 7. LL6=?
-   `(a)` `left = left->next`
-   `(b)` `right = right->next`
-   `(c)` `left = right->next`
-   `(d)` `right = left->next`

The logic is just like question 4, so the answer is (b).


> [!info]
> 延伸問題:
> 1.  解釋上述程式運作原理;
> 2.  指出程式改進空間，特別是考慮到 [Optimizing merge sort](https://en.wikipedia.org/wiki/Merge_sort#Optimizing_merge_sort);
> 3.  將上述 singly-linked list 擴充為 circular doubly-linked list 並重新實作對應的 `sort`;
> 4.  依循 Linux 核心 [include/linux/list.h](https://github.com/torvalds/linux/blob/master/include/linux/list.h) 程式碼的方式，改寫上述排序程式;
> 5.  嘗試將原本遞迴的程式改寫為 iterative 版本;

1. 

2. Since in the beginning of the function, the separating process only cuts the first node as `left` and the rest as `right`, `left` will not be separated anymore for the following recursion, while `right` will go N recursion, making the time complexity to be $\mathcal{O}(n^2)$ . If the separating process cuts the linked list from the center, the time complexity would reduce to $\mathcal{O}(n\log{} n)$ 

3. The revised version is [here](https://github.com/hfyeh/c-review/blob/master/linked_list_quiz2_circular_doubly_linked_list.c).

4. 

5. The iterative version is ...
