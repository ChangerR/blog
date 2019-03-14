---
title: 二叉树的遍历 
tags:
  - 算法
categories:
  - algorithm
date: 2015-03-22 16:59:19
---
提高二叉树，我们最先想到的是二叉树的遍历，二叉树的遍历正常有三种方式，分别为前序遍历，中序遍历，和后序遍历。

## 前序遍历(递归)

``` c
void treeWalk(node* n)
{
    if (n == NULL)
        return;
    visit(n);
    treeWalk(n->left);
    treeWalk(n->right);
}
```

## 中序遍历（递归）

``` c
void treeWalk(node* n)
{
    if (n == NULL)
        return;
    treeWalk(n->left);
    visit(n);
    treeWalk(n->right);
}
```

## 后序遍历（递归）

``` c
void treeWalk(node* n)
{
    if (n == NULL)
        return;
    treeWalk(n->left);
    treeWalk(n->right);
    visit(n);
}
```

## 前序遍历（迭代）

``` c

void treeWalk(node* n)
{
    Stack s;
    s.push(n);

    while(!s.empty())
    {
        node* t = s.pop()
        visit(t);
        s.push(t->right);
        s.push(t->left);
    }
}

```

## 中序遍历（迭代）

``` c

void treeWalk(node* n)
{
    Stack s;
    node* cur = n;
    
    while( cur != NULL || !s.empty())
    {
        while(cur != NULL)
        {
            s.push(cur);
            cur = cur->left;
        }

        cur = s.pop();
        visit(cur);
        cur = cur->right;
        
    }
}
```

## 后续遍历（迭代）

``` c

void treeWalk(node* n)
{
    Stack s;
    node* cur = n;
    node* last = NULL; //使用last指针标识右子节点是否已经访问过

    while(cur != NULL || !s.empty())
    {
        while(cur != NULL)
        {
            s.push(cur);
            cur = cur->left;
        }

        cur = s.top();

        if (cur->right != NULL && cur->right != last)
        {
            cur = cur->right;
        }
        else
        {
            visit(cur);
            last = cur;
            s.pop();
            cur = NULL;
        }
    }
}
```