---
title: "从数组中获取指定数量的随机元素"
date: 2019-03-13T20:41:18+08:00
description: "算法-从数组中获取指定数量的随机元素"
Tags: 
- 算法
- 数组
Categories:
- Java基础算法
---

### 问题
从给定的一个整型数组中，随机获取指定数量的数组元素。

### 思路一
新建一个与整型数组相同长度的boolean类型数组用来做标志位，标志位值为true表示当前元素是否已经获取，如果数组元素未被获取，则取出该元素，同时把对应的标志位置位true，如果发现当前元素已经获取，则重新随机获取。

```Java
public static int[] get(int[] array, int num) {
        int[] res = new int[num];
        if (array == null || array.length == 0 || num <= 0) {
            return res;
        }
        boolean[] used = new boolean[array.length];
        int i = 0;
        Random r = new Random();
        while(i < num) {
            int index = r.nextInt(array.length);
            if (!used[index]) {
                used[index] = true;
                res[i] = array[index];
                i++;
            }
        }
        return res;
    }
```
该算法牺牲了内存空间(创建了数组used), 同时随着num参数接近array长度效率会逐渐降低。

### 思路二

将每次取出的元素与末位元素交换位置，每次生成的<font color="red">index</font>下标逐渐缩小范围。

```Java
public static int[] optimizeGet(int[] array, int num) {
        int[] res = new int[num];
        if (array == null || array.length == 0 || num <= 0) {
            return res;
        }
        Random r = new Random();
        for (int i=0;i<num;i++) {
            int index = r.nextInt(array.length - i);
            res[i] = array[index];
            array[index] = array[array.length - i - 1];
            array[array.length - i - 1] = res[i];
        }
        return res;
    }
```

完整demo

```Java
public class GetRandNumbersFromArray {


    public static int[] get(int[] array, int num) {
        int[] res = new int[num];
        if (array == null || array.length == 0 || num <= 0) {
            return res;
        }
        boolean[] used = new boolean[array.length];

        int i = 0;
        int seq = 0;
        Random r = new Random();
        while(i < num) {
            System.out.println(seq++);
            int index = r.nextInt(array.length);
            if (!used[index]) {
                used[index] = true;
                res[i] = array[index];
                i++;
            }
        }
        return res;
    }

    public static int[] optimizeGet(int[] array, int num) {
        int[] res = new int[num];
        if (array == null || array.length == 0 || num <= 0) {
            return res;
        }
        Random r = new Random();
        for (int i=0;i<num;i++) {
            int index = r.nextInt(array.length - i);
            res[i] = array[index];
            array[index] = array[array.length - i - 1];
            array[array.length - i - 1] = res[i];
        }
        return res;
    }

    public static void main(String[] args) {
        int[] array = new int[]{1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 15, 16, 17, 18, 19, 20};
        int[] res = get(array, 19);
        Arrays.sort(res);
        System.out.println(Arrays.toString(res));
    }

}
```
