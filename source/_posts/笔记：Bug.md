---
title: 笔记：Bug
date: 2025-05-29
categories:
  - 文章类别
tags: []
author: 霸天
layout: post
---













## 断点调试模式（DEBUG）

### 准备代码

```  
public class TestDebug {  
    public static void main(String[] args) {  
        int [] arr = {5, 7, 6, 1, 3, 4, 2, 8, 6, 9};  
        sort(arr);  
        System.out.println(Arrays.toString(arr));  
        int i = 10;  
        i++;  
        int b = 1;  
        int c = b--;  
        System.out.println(i / c);  
    }  
  
    public static void sort(int[] arr) {  
        for (int i = 0; i < arr.length - 1; i++) {  
            for (int j = 0; j < arr.length - 1 - i; j++) {  
                if (arr[j] > arr[j + 1]) {  
                    int temp = arr[j];  
                    arr[j] = arr[j + 1];  
                    arr[j + 1] = temp;  
                }  
            }  
        }  
    }  
}
```

---


### 进入 DEBUG 模式

![](image-20250529094450634.png)

---


### DEBUG 基本功能
![](image-20250529095444405.png)

----




