---
title: "字符串的最小包含子串"
date: 2019-03-16T13:19:14+08:00
description: "算法-求解字符串的最小包含子串"
Tags:
- 算法
- 字符串
Categories:
- Java基础算法
---

### 问题
给定字符串str1，str2，获取字符串str1中包含str2的最小字符子串。

> * str1="abcde", str2="bd" -> bcd
> * str1="abcde", str2="cg" -> ""

### 思路
假定字符编码范围0~255

> * 创建一个size为256的整形数组charCount，用来保存字符串str2所有字符的出现次数
> * 整形变量match用来表示当前差几个字符未匹配
> * 将str1、str2分别转换为字符数组parentArray、subArray
> * 遍历subArray，保存str2每个字符出现的次数
> * 从左向右遍历parentArray，将每个字符对应的次数做--运算，之后如果字符的次数为零，表示匹配上一个字符，则match--
> * 若match等于0，表示str2中的所有字符均已匹配上，此时parentArray的字符位置则是最小子串的右边界限
> * 开始确定最小子串的左边界限，再次遍历parentArray，查找第一次出现次数为-1的字符位置，该位置则为左边界限
> * 获取最小子串str1.subString(左边界+1,右边界+1)

### 代码

```Java
**
 * 获取最小包含的子串
 * str1="some3gs", str2="og" -> ome3g
 */
public class GetMinContainsSubString {

    public static String getMinSubString(String str1, String str2) {
        if (str1 == null || str2 == null || str2.length() > str1.length()) {
            return "";
        }
        int[] charCount = new int[256];
        char[] parentArray = str1.toCharArray();
        char[] subArray = str2.toCharArray();
        for (int i=0;i<subArray.length;i++) {
            charCount[subArray[i]]++;
        }
        //最小子串右边界位置
        int i = 0;
        //最小子串左边界位置
        int j = 0;
        //需要匹配的总字符数
        int match = str2.length();
        String minSubString = "";
        while(i != parentArray.length) {
            char c = parentArray[i];
            charCount[c]--;
            //减1之后还大于等于0,表示之前存在该字符
            if (charCount[c] >= 0) {
                match--;
            }
            //此时已全部匹配,i则是右边界位置，此时还需压缩左边界位置
            if (match == 0) {
                while(charCount[parentArray[j++]] == -1 ) {
                    minSubString = str1.substring(j + 1, i + 1);
                    return minSubString;
                }
            }
            i++;
        }
        return minSubString;
    }

    public static void main(String[] args) {
        String str1 = "some45baby";
        String str2 = "m4a";
        System.out.println(getMinSubString(str1, str2));
    }
}
```