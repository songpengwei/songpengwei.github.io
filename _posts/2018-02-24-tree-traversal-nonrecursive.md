---
title: 数据结构与算法（一）：二叉树的非递归遍历
updated: 2018-02-24 00:34
tag: tree, data structures, algorithms
---

## 概述
最近在琢磨关于树的非递归遍历的一些思路，和对应的实现，写在这里，聊以备忘。 

## 直接迭代写法
基本思路是，一路往左走，撞了南墙（碰到NULL）就回头；回头摆到右（子树的根），接着往左走（循环）。

基本代码如下：

```C++
/**
 * Definition for a binary tree node.
 * struct TreeNode {
 *     int val;
 *     TreeNode *left;
 *     TreeNode *right;
 *     TreeNode(int x) : val(x), left(NULL), right(NULL) {}
 * };
 */
void traversal(TreeNode* root) {
    stack<TreeNode*> s;
    TreeNode* current = root;

    while (!s.empty() || current) {
        while (current) {
            s.push(current); //(1)
            current = current->left;
        }

        current = s.top()->right; // (3)
        s.pop(); // if we pop the node, then we do not have (5)
    }

    return ans;
}
```


对应的完整遍历路径如下：

```
      (1) root (5?)
         / (3)  \
        /        \
(2)l-subtree   (4)r-subtree
```
对于某个节点root来说，有三次访问机会：

1. 从父亲节点过来(1)
2. 访问完左子树后回来(3)
3. 访问完右子树后回来(5)

对应的，按照二叉树遍历的定义，

1. 如果在(1)访问节点，就是前序遍历：先根(1)，然后左子树(2)，最后右子树(4)。
2. 如果在(3)访问节点，就是中序遍历：先左子树(2)，然后根(3)，最后右子树(4)。
3. 如果在(5)访问节点，就是后序遍历：先左子树(2)，然后右子树(4)，最后根(5)。

由于访问完左子树需要继续访问右子树，故需要保存root的指针，即stack在这里的作用。随着访问的深入，stack不断入栈节点，最大长度即为树的深度。这里需要着重说明的是，随着访问左子树归来，将会不断的退栈，因为在获得右子树指针后，已经不需要再保存root的指针了。因此如果不做特殊处理，步骤(5)是没有的。

所以对于二叉树前序遍历和中序遍历的非递归代码，可以直接给出：


```C++
// preorder traversal:
void preorderTraversal(TreeNode* root) {
    stack<TreeNode*> s;
    TreeNode* current = root;

    while (!s.empty() || current) {
        while (current) {
            s.push(current); 
            cout << current->val << endl;//(1)
            current = current->left;
        }

        current = s.top()->right; 
        s.pop();
    }

    return ans;
}

// inorder traversal:
void inorderTraversal(TreeNode* root) {
    stack<TreeNode*> s;
    TreeNode* current = root;

    while (!s.empty() || current) {
        while (current) {
            s.push(current);
            current = current->left;
        }

        current = s.top()->right; 
        cout << s.top()->val << endl;//(3)
        s.pop();
    }

    return ans;
}
```
对于后序遍历的代码，会稍微复杂一点，所需要做的改动主要为：不能在第二次访问（即图中(3)）后退栈，而需要在阶段(5)退栈。这就牵扯出如何判断阶段(5)的问题。这里我的做法是引入一个prev指针，标记访问序列中前一个二叉树节点，如果root->right即为prev，或者root->right为NULL，就可以判断已经从右子树访问返回，即为阶段(5)。

```C++
void postorderTraversal(TreeNode* root) {
    stack<TreeNode*> s;
    TreeNode* current = root, *prev = NULL;

    while (current || !s.empty()) {
        while (current) {
            s.push(current);
            current = current->left;
        }

        TreeNode* now = s.top();
        if (now->right == NULL || now->right == prev) {
            // stage (5)
            cout << now->val;
            prev = now; // update the prev
            s.pop();
            current = NULL; // let the stack keep popping
        } else {
            // stage (4)
            current = now->right;
        }
    }

    return ans;
}
```

## 递归翻译写法
我们知道，可以用栈来模拟函数调用，或者说函数调用本来就是函数栈帧的入栈和退栈。由于二叉树遍历足够简单，也让我们的模拟变的相对容易实现。

比如，对于先序遍历，由于栈会使访问反序，因此先压入右子树。

```C++
void preorderTraversal(TreeNode* root) {
    stack<TreeNode*> s;
    if (root) s.push(root);
    while (!s.empty()) {
        TreeNode* now = s.top();
        cout << now->val;
        s.pop();

        if (now->right) s.push(now->right);
        if (now->left) s.push(now->left);
    }

    return ans;
}
```
如果先压入左子树呢，就成了逆后序遍历，这样只需要再加一个栈，就能得到后序遍历。
```C++
void inversePostorderTraversal(TreeNode* root) {
    stack<TreeNode*> s;
    if (root) s.push(root);
    while (!s.empty()) {
        TreeNode* now = s.top();
        cout << now->val;
        s.pop();

        if (now->left) s.push(now->left);
        if (now->right) s.push(now->right);
    }

    return ans;
}
```

## Morris中序遍历
以上遍历方法，不论递归还是非递归，其额外的空间复杂度都为O(h)即O(lgn)，因为栈开销最大为树的深度。那么有没有一种不借助额外空间的方法来实现树的遍历呢？聪明的你可能会想到线索二叉树，Bingo！这就是Morris中序遍历。

闲话少说，先上代码：

```C++
void inorderTraversal(TreeNode* root) {
    TreeNode* current = root;
    while (current != NULL) {
        if (current->left == NULL) {
            cout << current->val;
            current = current->right;
        } else {
            TreeNode* prev = current->left;
            while (prev->right != NULL && prev->right != current) {
                prev = prev->right;
            }

            if (prev->right == NULL) {
                prev->right = current;
                current = current->left;
            } else if (prev->right == current) {
                cout << current->val;
                current = current->right;
                prev->right = NULL;
            }
        }
    }
}
```
基本思路是：


```
1. Initialize current as root 
2. While current is not NULL
   If current does not have left child
      a) Print current’s data
      b) Go to the right, i.e., current = current->right
   Else
      a) Make current as right child of the rightmost node in current's left subtree
      b) Go to this left child, i.e., current = current->left

```
好吧，其实这就是伪代码，出处见[这里](https://www.geeksforgeeks.org/inorder-tree-traversal-without-recursion-and-without-stack/)。核心要点就是，当我们一头扎向南墙时，为了能无痛返回，需要在南墙打个洞，通回原路，并且过了洞之后将洞补上。但是打洞是需要时间的，这就是典型的用时间换空间--每次都得在找左孩子最右边那个元素的时候多浪费点时间。

## 小结
有些点还没想清楚，以后再补充。









     









