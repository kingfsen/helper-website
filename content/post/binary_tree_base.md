---
title: "二叉树"
date: 2019-03-23T09:55:45+08:00
description: "二叉树(二叉搜索树)的用途与实现"
thumbnail: "img/binary.png"
Tags:
- 二叉树
- 数据结构
Categories:
- Java基础算法
---
先简单了解下有序数组和链表两种数据结构

- 有序数组<br/>
    优点：用二分查找法可以在有序数组中快速查找特定的值，时间O(logN)，当然按顺序遍历也只是O(N)<br/>
    缺点：插入或者删除，需要多次移动数据项，平均要移动N/2次，不适合发生很多插入或者删除操作的场景
  
- 链表<br/>
    优点：链表的插入和删除都非常快，时间O(1)<br/>
    缺点：查找数据必须从头开始，平均查找N/2个数据，时间趋近O(N)

二叉树结合了有序数组和链表的优点，在树中查找数据和有序数组一样快，并且插入和删除数据和链表的速度也和链表一样。

---

### 基本概念

![binarytree](/blog/binary_tree_base/001.png)

树的定义是从根节点到其他任何一个节点有且只有一条路径，根据树的定义，下图展示的并不是一颗正确的树。

![nottree](/blog/binary_tree_base/002.png)

- 路径，从节点到另一个节点，所经过的节点顺序排列即为路径
- 根，树顶端的节点即为根，一棵树只有一个根
- 父节点，除了根之外的节点都有一条边向上连接到另一个节点，上面这个节点即为下面节点的父节点
- 子节点，每个节点都有一条或者多条边向下连接到其他节点，这些节点即为子节点
- 叶节点，没有子节点的节点即为叶子节点
- 子树，每个子节点都可以作为子树的根，它和它的子节点构成了子树
- 关键字，节点中存放的数据域
- 层，从根到当前节点的代数，根为0，子节点则为1，孙子节点层即为2
- 二叉树，每个节点有且仅有最多两个节点，这样的树即为二叉树
- 搜索二叉树，节点的左子节点的关键字值小于该节点，右子节点的关键字值大于该节点

---

### 二叉搜索树

二叉树是一种特殊的树，它的每个节点最多有两个子节点。有些树是非平衡的，即它的大部分节点在根的一边，个别的子树也可能是非平衡的。
树变得不平衡，是由于插入顺序导致的，如果插入序列是升序，则所有的值都是右节点，如果插入序列是降序，则所有的值都是左节点。

二叉搜索树是二叉树中的应用广泛的其中一种，常见二叉搜索树基本操作。
首先定义树中的节点Node

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

树的抽象结构，它只有一个根

```Java
public class Tree {
    public Node root;

    public Tree() {
    
    }
 }
```
---
- 插入<br/>
      插入方法insert()是构造树的必备方法，新节点插入必须首先找到待插入的位置。从根节点开始搜索，若节点的值小于待插入值，则搜索子左子树，否则搜索右子树，如此循环遍历，
      最后必定找出节点为null的位置，即为待插入位置。
      
      ```Java
      //insert one new node
    public void insert(int data) {
        Node node = new Node(data);
        if (root == null) {
            root = node;
        } else {
            Node current = root;
            Node parent;
            while (true) {
                parent = current;
                if (current.data > data) {
                    current = current.left;
                    if (current == null) {
                        parent.left = node;
                        return;
                    }
                } else {
                    current = current.right;
                    if (current == null) {
                        parent.right = node;
                        return;
                    }
                }
            }
        }
    }
      ```

- 查找<br/>
      查找也只能从根节点开始搜索，查找节点的时间取决于这个节点所在的层，一个5层的二叉树最多节点数N为1+2+4+8+16=31，最多需要搜索5次，时间复杂度接近O(log<sub>2</sub>N)。
      
      ```Java
      public Node find(int data) {
        Node current = root;
        while (current.data != data) {
            if (current.data > data) {
                current = current.left;
            } else {
                current = current.right;
            }
            if (current == null) {
                return null;
            }
        }
        return current;
    }
      ```
- 最大最小值<br/>
      查找二叉搜索树中最大最小节点，非常简单。从根节点开始，最大节点只需一直查找右子节点，最小节点只需一直查找左子树。
      
      ```Java
      public Node findMax() {
        Node current = root;
        Node parent = null;
        while (current != null) {
            parent = current;
            current = current.right;
        }
        return parent;
    }
    
    public Node findMin() {
        Node current = root;
        Node parent = null;
        while (current != null) {
            parent = current;
            current = current.left;
        }
        return parent;
    }
      ```

- 删除<br/>
      删除是二叉搜索树中最复杂的一个操作，也是很基本的常见操作。删除节点，首先也要从根节点开始进行查找，找到节点之后，需要考虑待删除节点的多种情况。
      1. 该节点是叶节点<br/>
          删除叶节点是最简单的，只需把该节点的父节点对应的字段置为null即可，待删除的节点即成为了孤立节点，被回收。
      
          ![noleafnode](/blog/binary_tree_base/003.png)
      
      2. 该节点有一个子节点<br/>
          只需把待删除节点的子节点连接到其父节点即可，通过改变待删除节点的父节点中的节点引用属性实现。
      
          ![oneleafnode](/blog/binary_tree_base/004.png)
      
      3. 该节点有两个子节点<br/>
          如果待删除的节点同时有左右子节点，这种情况就比较复杂，不能直接简单剔除节点，用子树代替。删除有2个子节点的节点，必须用它的中序后继节点来替代该节点。
      
          ![twoleafnode](/blog/binary_tree_base/006.png)
      
          如果待删除节点的后继节点也有自己的子节点，那么这种情况更麻烦，此时必须找到中序后继节点的子树中最小的一个节点。所以整个思路归结为当待删除节点为A时，
          必须找到关键值比A节点值大的节点集合中最小的一个节点，也就是最接近A节点值的节点来代替其位置。以A节点右子节点为根节点的子树，
          一直查找左子节点，最后一个左子节点即为最小节点，如果A节点右子节点无左子节点，那么A节点右子节点即为最小节点。
      
          ![twoleafnode1](/blog/binary_tree_base/007.png)
          
          上图中左边二叉搜索树中待删除节点为38，它的后继节点即为41，右边二叉搜索树中待删除节点38，其右子节点72并无左子节点，因此72即为38节点的后继节点。
          
          查找后继节点代码实现，找到之后，调整各节点之间引用关系，设置节点55的左子节点为43，节点41的右子节点不再是41，而是待删除节点38的右子节点的72。
          
          ```Java
          public Node findSuccessor(Node node) {
              Node successorParent = node;
              Node successor = node;
              Node current = node.right;
              while (current != null) {
                  successorParent = successor;
                  successor = current;
                  current = current.left;
              }
              if (successor != node.right) {
                  successorParent.left = successor.right;
                  successor.right = node.right;
              }
              return successor;
          }
          ```
          
          综合上述三种情况，删除代码实现，请注意判断待删除节点是否是根节点
          
          ```Java
          public boolean delete (int data) {
              Node current = root;
              Node parent = root;
              boolean left = false;
              while (current.data != data) {
                  parent = current;
                  if (current.data > data) {
                      current = current.left;
                      left = true;
                  } else {
                      current = current.right;
                      left = false;
                  }
                  //not find special node
                  if (current == null) {
                      return false;
                  }
              }
              //leaf node
              if (current.left == null && current.right == null) {
                  //maybe want to delete the whole tree
                  if (current == root) {
                      root = null;
                  } else if (left) {
                      parent.left = null;
                  } else {
                      parent.right = null;
                  }
              //one children node
              } else if (current.right == null) {
                  if (current == root) {
                      root = current.left;
                  } else if (left) {
                      parent.left = current.left;
                  } else {
                      parent.right = current.left;
                  }
              } else if (current.left == null) {
                  if (current == root) {
                      root = current.right;
                  } else if (left) {
                      parent.left = current.right;
                  } else {
                      parent.right = current.right;
                  }
              //two children node
              } else {
                  Node successor = findSuccessor(current);
                  if (current == root) {
                      root = successor;
                  } else if (left) {
                      parent.left = successor;
                  } else {
                      parent.right = successor;
                  }
                  successor.left = current.left;
              }
              return true;
          }
          ```
          由于删除的逻辑比较复杂，更多的时候我们通过在Node节点中通过增加一个boolean属性deleted来表示当前节点是否逻辑上删除，并不用真正的去进行删除，
          find查找的时候只要过滤掉deleted=true的节点数据。
          
---

### 数组表示树

树的结构完全可以用数组来表示，节点数据存入数组中，而不是由左右引用相连，通过数组的下标来维护其在树中的位置。

![array](/blog/binary_tree_base/008.png)

树中的每个位置不论是否有节点，都对应数组中的一个元素，那么数组中没有的数据元素为0或者null，表示树在此位置无节点。
寻找树中节点的子节点或者父节点可以通过算术表达式来计算数组索引值，比如某节点的索引值为n，那么它相关联的节点索引如下。

- 左子节点，2 * n + 1
- 右子节点，2 * n + 2
- 父节点，(n - 1) / 2

当二叉树不是一颗满树的时候，用数组表示会浪费很多空间，同时如果树具备删除操作，需要移动子树时，会造成大量数组元素的移动，效率不是很高。

### 关键值重复

整个二叉搜索树的构造过程中，请保持没有重复值，可以在insert的时候先调用find进行查询一次，阻止重复数据的节点加入树中。