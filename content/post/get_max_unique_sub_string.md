---
title: "寻找字符串中不重复最长子串"
date: 2019-03-14T15:32:46+08:00
description: "算法-寻找字符串中不重复最长子串"
Categories: 
- Java基础算法
Tags:
- 字符串
---

### 问题

给定一个字符串，找出这个字符串中最长的不重复子串。假定字符串编码范围在256之内(排除中文等特殊字符)，同时如果有相同长度的子串，优先获取首次寻找的子串，时间复杂度O(N)。

> * "abcd" -> "abcd"
> * "abccd" -> "abc"
> * "somok39ebab3yuvwz123" -> "ab3yuvwz12"

### 思路

- 用一个int数组position保存每个字符在字符串中的位置
- 用一个int变量mark标记下一次可获取子串的开始位置
- 用一个String变量maxUniqueStr保存循环过程中始终最长的无重复子串。


> * 初始化position中每个元素值为-1，同时将字符串转化成字符数组chars
> * 遍历chars数组，当前字符上次出现，取出上次的位置lastPos(注意mark必须小于lastPos)
> * 获取子串, mark到当前遍历的序号i之间的子串，子串大于maxUniqueStr，则将子串保存到maxUniqueStr
> * mark向前推进为当前字符的上次出现位置加1，即lastPos + 1
> * 处理循环之外的最后尾部子串逻辑

### 代码实现

```Java
public class GetMaxUniqueSubString {


    public static String getMaxUniqueString(String str) {
        if (str == null || str.length() == 0 ) {
            return "";
        }
        //循环过程中保存的最长字符串
        String maxUniqueStr = "";
        //标记位
        int mark = 0;
        char[] chars = str.toCharArray();
        //存放字符的位置
        int[] position = new int[256];
        //位置都初始化-1,表示未发现
        for (int i=0;i<position.length;i++) {
            position[i] = -1;
        }
        for (int i=0;i<chars.length;i++) {
            //当前字符的上一次出现位置
            int lastPos = position[chars[i]];
            //不为-1表示之前出现过该字符
            if (lastPos != -1 && mark <= lastPos) {
                String subStr = str.substring(mark, i);
                if (subStr.length() > maxUniqueStr.length()) {
                    maxUniqueStr = subStr;
                }
                mark = lastPos + 1;
            }
            position[chars[i]] = i;
        }
        //可能漏掉结尾一段
        if (mark <= chars.length) {
            String subStr = str.substring(mark, chars.length);
            if (subStr.length() > maxUniqueStr.length()) {
                maxUniqueStr = subStr;
            }
        }

        return maxUniqueStr;
    }

    public static void main(String[] args) {
        String str = "abccd";
        String maxUniqueString = getMaxUniqueString(str);
        System.out.println(maxUniqueString);
    }
}
```
