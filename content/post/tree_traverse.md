---
title: "递归与非递归遍历二叉树"
date: 2019-03-24T16:01:34+08:00
thumbnail: "img/traverse.png"
description: "二叉树的递归遍历与非递归遍历详解"
Tags:
- 二叉树
- 数据结构
Categories:
- Java基础算法
---

二叉树的遍历有三种方法，分别是先序、中序、后序，先序遍历顺序为根、左、右，中序遍历顺序为左、根、右，后序遍历顺序为左、右、根。
遍历二叉树的方式又包括递归、非递归两种方式。

![tree](/blog/tree_traverse/tree.png)

- 先序遍历结果：50、30、20、40、60
- 中序遍历结果：20、30、40、50、60
- 后序遍历结果：20、40、30、60、50

树节点Node对象结构

```Java
class Node {
    public Node left;
    public Node right;
    public int data;

    public Node() {

    }

    public Node(int data) {
        this.data = data;
    }

    public boolean equals(Node node) {
        if (node == null) {
            return false;
        }
        return node.data == this.data;
    }
}
```

### 递归

递归遍历二叉树代码实现相对比较简单

- 先序遍历

    ```Java
    public void preOrder(Node root) {
            if (root == null) {
                return;
            }
            System.out.println(root.data);
            preOrder(root.left);
            preOrder(root.right);
        }
    ```

- 中序遍历

    ```Java
    public void inOrder(Node root) {
        if (root == null) {
            return;
        }
        inOrder(root.left);
        System.out.println(root.data);
        inOrder(root.right);
    }
    ```
    
- 后序遍历

    ```Java
    public void postOrder(Node root) {
        if (root == null) {
            return;
        }
        postOrder(root.left);
        postOrder(root.right);
        System.out.println(root.data);
    }
    ```
    
### 非递归
用递归方法解决的问题都能用非递归的方法去实现，因为递归方法无非是利用了系统函数栈来保存中间结果信息，
非递归方式则完全由我们自己选择数据结构来代替函数栈，实现相同功能。

- 先序遍历

    创建一个Stack来保存节点，首先把根节点压入栈中，循环判断Stack是否为空，不为空，弹出节点，然后把节点的右子节点压入栈中，接着把节点的左子节点压入栈中。
    
    - 首先将节点50入栈
    - 弹出栈中节点50，打印其值
    - 节点50右子节点不为空，将右子节点压入栈中
    - 节点50左子节点不为空，将左子节点压入栈中
    - 弹出栈中节点，依照第二步逻辑循环处理节点
    
    ```Java
    public void preOrderUnRecur(Node root) {
        if (root == null) {
            return;
        }
        Stack<Node> stack = new Stack<>();
        stack.add(root);
        while (!stack.empty()) {
            Node node = stack.pop();
            System.out.println(node.data);
            if (node.right != null) {
                stack.push(node.right);
            }
            if (node.left != null) {
                stack.push(node.left);
            }
        }
    }
    ```
    
- 中序遍历

    创建一个Stack来保存节点，先依次将节点的左子节点压入栈中，当左子节点为空时，弹出栈中节点，将节点的右子节点压入栈中，循环处理。
    - 将节点50、30、20依次入栈
    - 节点20无左字节点，则弹出节点20，打印其值
    - 将root设置为20节点的右子节点
    - 20右子节点为空，继续弹出节点30
    - 将节点30的右子节点作为root，依照上面的逻辑循环处理
    
    ```Java
    public void inOrderUnRecur(Node root) {
        if (root == null) {
            return;
        }
        Stack<Node> stack = new Stack<>();
        while (root != null || !stack.empty()) {
            if (root != null) {
                stack.push(root);
                root = root.left;
            } else {
                Node node = stack.pop();
                System.out.println(node.data);
                root = root.right;
            }
        }
    }
    ```
    
- 后序遍历

    方式一：申请两个栈来保存节点
    
    ```Java
    public void postOrderUnRecur(Node root) {
        if (root == null) {
            return;
        }
        Stack<Node> s1 = new Stack<>();
        Stack<Node> s2 = new Stack<>();
        s1.push(root);
        while (!s1.empty()) {
            root = s1.pop();
            if (root.left != null) {
                s1.push(root.left);
            }
            if (root.right != null) {
                s1.push(root.right);
            }
        }
        while (!s2.empty()) {
            System.out.println(s2.pop().data);
        }
    }
    ```
    
    方式二：只申请一个栈保存节点
    
    ```Java
    public void postOrderUnRecurByOneStack(Node root) {
        if (root == null) {
            return;
        }
        Stack<Node> stack = new Stack<>();
        stack.push(root);
        Node node = null;
        while (!stack.empty()) {
            node = stack.peek();
            if (node.left != null && root != node.left && root != node.right) {
                stack.push(node.left);
            } else if (node.right != null && root != node.right) {
                stack.push(node.right);
            } else {
                System.out.println(stack.pop());
                root = node;
            }
        }
    }
    ```