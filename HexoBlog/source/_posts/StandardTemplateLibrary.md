---
title: Standard Template Library
categories: Cpp
date: 2024-05-09
updated: 2026-04-02
---
# 标准模板库容器

|容器|数据结构|时间复杂度|元素是否有序|元素是否可重复|注意事项|
|---|---|---|---|---|---|
|Array|数组|$O(1)$|无序|可重复|支持任意访问|
|Vector|数组|在尾部操作时$O(1)$，在头部操作时$O(n)$|无序|可重复|支持任意访问|
|Deque|双端队列|$O(1)$|无序|可重复|支持任意访问|
|forward_list|单向链表|$O(1)$|无序|可重复|不支持任意访问|
|List|双向链表|$O(1)$|无序|可重复|不支持任意访问|
|Stack|基于链表，deque/lsit|$O(1)$|无序|可重复|deque/list封闭头端开口。|
|Queue|基于链表，deque/list|$O(1)$|无序|可重复|deque/list封闭头端开口。|
|priority_queue|vector+max_heap|$O(log_2n)$|有序|可重复|使用堆处理和规则的vector容器|
|Set|红黑树|$O(log_2n)$|有序|不可重复|\|
|Multiset|红黑树|$O(log_2n)$|有序|可重复|\|
|Map|红黑树|$O(log_2n)$|有序|不可重复|\|
|Multimap|红黑树|$O(log_2n)$|有序|可重复|\|
|unordered_set|哈希表|$O(1)-O(n)$|无序|不可重复|\|
|unordered_multiset|哈希表|$O(1)-O(n)$|无序|可重复|\|
|unordered_map|哈希表|$O(1)-O(n)$|无序|不可重复|\|
|unordered_multimap|哈希表|$O(1)-O(n)$|无序|可重复|\|