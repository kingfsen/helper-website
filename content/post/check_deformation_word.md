---
title: "判断两个单词是否为变形词"
date: 2019-03-16T11:29:21+08:00
description: "算法-判断两个单词是否为变形词"
Tags:
- 字符串
- 算法
- 变形词
Categories:
- Java基础算法
---

### 问题
给定两个单词word1，word2，判断两个单词是否是变形词，即两个单词中的每个字符出现的次数一致。

> * word1="abcc", word="acbc", 返回true
> * word1="abcc", word="abbc", 返回false
> * word1="abc", word="cba", 返回true

### 思路
假定字符串的编码范围0~255。

> * 新建一个size为256的int数组
> * 分别将单词word1、word2转换为对应的字符数组word1CharArray、word2CharArray
> * 遍历word1CharArray，将字符出现的字符保存在int数组中
> * 遍历word2CharArray，将int数组中对应字符位置的值减1，如果减1之后值小于0，直接返回false

### 代码

```Java
/**
 * 判断两个单词是否是变形词
 */
public class CheckDeformationWord {

    public static boolean deformationWord(String word1, String word2) {
        if (word1 == null || word2 == null || word1.length() != word2.length()) {
            return false;
        }
        int[] charCount = new int[256];
        char[] word1ChartArray = word1.toCharArray();
        char[] word2ChartArray = word2.toCharArray();
        for (int i=0;i<word1ChartArray.length;i++) {
            charCount[word1ChartArray[i]]++;
        }
        for (int i=0;i<word2ChartArray.length;i++) {
            char c = word2ChartArray[i];
            charCount[c]--;
            if (charCount[c] < 0) {
                return false;
            }
        }
        return true;
    }

    public static void main(String[] args) {
        String word1 = "regher";
        String word2 = "reehgr";
        System.out.println(deformationWord(word1, word2));

    }
}
```

