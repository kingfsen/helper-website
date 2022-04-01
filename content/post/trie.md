---
title: "字典树的应用与实现"
date: 2019-03-16T19:00:11+08:00
thumbnail: "img/avatar.png"
description: "字典树、前缀树的实现与用途详解"
Tags:
- 字典树
- 算法
Categories:
- Java基础算法
---

字典树又称为单词查找树或者前缀树，是一种用于快速检索的树形结构，比如小写字母词典数是一个26叉数，数字的字典树是一个10叉数。字典数的键并未保存在节点中，而是由节点在树中的位置决定的。
根节点一般对应空信息。字典树的优点是查询效率高，其核心思想是利用空间换时间，利用字符串的公共前缀来提高效率。

![trie](/blog/trie/001.png)

基本性质

- 根节点没有字符路径，除根节点外，每一个节点都被一个字符路径找到
- 从根节点到某一个节点，将路径上经过的字符连接起来，为扫过的对应字符串
- 每个节点向下所有的字符路径上的字符都不同

特点

- 插入、查找的时间复杂度均为O(N)，其中N为字符串长度
- 空间复杂度是26^n级别的，非常庞大，不适用超长字符编码范围

用途

> * 字符串检索，从根节点开始一个一个字符进行比较，所有的字符全部比较并且完全相同，还需要判断最后一个节点标识位
> * 字符串排序，遍历一次所有关键字，将它们全部插入trie树，树的每个结点的所有儿子很显然地按照字母表排序，然后先序遍历输出Trie树中所有关键字即可
> * 字符串最长公共前缀
> * 词频统计，常被搜索引擎用于文本词频统计
> * 字符串搜索的前缀匹配，用于搜索提示非常方便

在字典树上搜索单词的步骤

- 从根节点开始搜索
- 取得要查找单词的第一个字符，并根据该字符对应的字符路径继续向下层继续搜索
- 一直向下搜索，如果单词搜索完，判断找到的最后这个节点的end属性值是否大于0，即是不是最终节点，如果不是表示当前字典树未收入该单词

节点中包含一个int字段path，用来记录有多少单词含有该字符，用一个int字段end来记录有多少单词以当前节点字符为尾缀。

```Java
/**
 * 字典树
 * A-Z ascii码范围65~90
 * a-z ascii码范围97-122
 * Assume all word characters range a~z
 */
public class Trie {

    private TrieNode root;

    public Trie() {
        root = new TrieNode();
    }

    public void insert(String word) {
        if (word == null || word.length() == 0) {
            return;
        }
        TrieNode node = root;
        int index;
        char[] chars = word.toCharArray();
        for (int i=0;i<chars.length;i++) {
            index = chars[i] - 'a';
            if (node.map[index] == null) {
                node.map[index] = new TrieNode();
            }
            node = node.map[index];
            node.path++;
        }
        node.end++;
    }

    public boolean search(String word) {
        if (word == null || word.length() == 0) {
            return false;
        }
        TrieNode node = root;
        int index;
        char[] chars = word.toCharArray();
        for (int i=0;i<chars.length;i++) {
            index = chars[i] - 'a';
            if (node.map[index] == null) {
                return false;
            }
            node = node.map[index];
        }
        return node.end != 0;
    }

    public void delete(String word) {
        if (word == null || word.length() == 0) {
            return;
        }
        TrieNode node = root;
        int index;
        char[] chars = word.toCharArray();
        for(int i=0;i<chars.length;i++) {
            index = chars[i] - 'a';
            if (node.map[index].path-- == 1) {
                node.map[index] = null;
                return;
            }
            node = node.map[index];
        }
        node.end--;
    }

    public int prefixCount(String pre) {
        if (pre == null || pre.length() == 0) {
            return 0;
        }
        TrieNode node = root;
        int index;
        char[] chars = pre.toCharArray();
        for (int i=0;i<chars.length;i++) {
            index = chars[i] - 'a';
            if (node.map[index] == null) {
                return 0;
            }
            node = node.map[index];
        }
        return node.path;
    }

    public static void main(String[] args) {
        Trie t = new Trie();
        t.insert("iloveyou");
        t.insert("iloveme");
        t.insert("iloveyouverymuch");
        t.insert("iloveyouwife");
        t.insert("youloveme");
        t.insert("youlovemesomuch");
        System.out.println(t.prefixCount("ilove"));
        t.delete("iloveme");
        System.out.println(t.prefixCount("ilove"));
    }

}

class TrieNode {

    public int path;

    public int end;

    public TrieNode[] map;

    public TrieNode() {
        map = new TrieNode[26];
    }
}
```




