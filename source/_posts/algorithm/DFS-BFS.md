---
title: 图的遍历
tags:
  - 算法
  - 图
categories:
  - algorithm
date: 2015-03-27 16:59:19
---
### 深度优先搜索
在DFS中，我们会访问节点r,然后循环访问r的每个相邻节点。在访问r的相邻节点n时，我们会在继续访问r的其他相邻结点之前先访问n的所有相邻节点。也就是说，在继续搜索r的其他子节点之前，我们会先穷尽搜索n的子节点。

``` c
void search(Node root)
{
    if (root == NULL)
        return;
    visit(root);
    root.visited = true;
    foreach(Node n in root.adjacent)
    {
        if (n.visited == false)
        {
            search(n);
        }
    }
}
```

### 广度优先搜索
在BFS中，我们会在搜索r的“孙子节点”之前先访问节点r的所以相邻节点。用队列实现的迭代方案往往最有效。

``` c
void search(Node root)
{
    Queue queue = new Queue();
    root.visited = true;
    visit(root);
    queue.enqueue(root);

    while(!queue.isEmpty())
    {
        Node r = queue.dequeue();

        foreach (Node n in r.adjacent)
        {
            if (n.visited == false)
            {
                visit(n);
                n.visited = true;
                queue.enqueue(n);
            }
        }
    }
}
```